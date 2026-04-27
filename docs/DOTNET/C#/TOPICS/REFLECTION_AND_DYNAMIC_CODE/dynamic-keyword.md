# The `dynamic` Keyword

_Targets .NET 10 / C# 14. See also: [Reflection and member discovery](./reflection-and-member-discovery.md), [Dynamic type creation and Activator](./dynamic-type-creation-and-activator-class.md)._

`dynamic` was introduced in C# 4 (2010) to make COM interop and IronPython hosting bearable. The compiler treats a `dynamic` expression as `object`-with-an-escape-hatch: type checking is deferred until runtime, when the **DLR** (Dynamic Language Runtime) resolves the call against whatever the value actually is.

In modern C#, `dynamic` is a niche tool. Pattern matching, generics, `JsonNode`/`JsonElement`, and source generators have eaten most of its use cases. But COM interop and a handful of duck-typed scenarios still call for it, and every senior should know exactly what the compiler emits — because the perf cliff and the AOT incompatibility are real.

## What `dynamic` actually is

At compile time, `dynamic` is `System.Object` decorated with `[Dynamic]` so the compiler remembers to wrap callsites:

```csharp
dynamic d = GetThing();
int n = d.Compute(42);
```

That second line lowers (roughly) to:

```csharp
// Synthesised callsite, cached in a static field.
static CallSite<Func<CallSite, object, int, object>> _site
    = CallSite<Func<CallSite, object, int, object>>.Create(
        Microsoft.CSharp.RuntimeBinder.Binder.InvokeMember(
            CSharpBinderFlags.None,
            "Compute",
            null,
            typeof(MyType),
            new[] { CSharpArgumentInfo.Create(...), CSharpArgumentInfo.Create(...) }));

int n = (int)_site.Target(_site, d, 42);
```

The `CallSite<T>` caches the resolved binding **per-callsite** in a polymorphic-inline-cache structure (L0/L1/L2) — the same idea as a JS engine's IC. First call is expensive (the binder walks reflection, builds an expression tree, compiles it). Subsequent calls hit the cache and run roughly **5×–10× faster than `MethodInfo.Invoke`**, but still **5×–50× slower than a direct call**.

Reference: `Microsoft.CSharp.RuntimeBinder.Binder` is what makes `dynamic` work for C# semantics; the BCL's `System.Dynamic` provides the runtime infrastructure.

## DLR vs reflection — the trade-off

| Concern                | Reflection (`Invoke`)                       | `dynamic` (DLR callsite)               |
|------------------------|---------------------------------------------|----------------------------------------|
| Per-call cost          | Always pays full lookup + boxing            | Pays full cost on first call, cached after |
| Caching                | You implement it                            | Built-in, per callsite                 |
| Argument shape         | `object[]`, value types boxed               | Strongly-typed delegate signature, value types unboxed once cache is warm |
| Method resolution      | You pick the `MethodInfo`                   | C# overload resolution rules at runtime |
| Operators (`+`, `==`)  | Hand-rolled                                 | Just works (`dynamic a + dynamic b`)   |
| Trimming / AOT         | Needs `[DynamicallyAccessedMembers]`        | Unsupported under NativeAOT — DLR depends on Reflection.Emit |

So `dynamic` is "reflection with a JIT-compiled, cached fast path that happens to follow C# language rules". When you want C# semantics applied at runtime — overload resolution, operator lookup, conversions — `dynamic` is dramatically more concise than hand-written reflection. When you want a single specific method called, raw reflection (cached as a delegate) is faster and AOT-tractable.

## Where it actually shines

- **COM interop.** `dynamic` was designed for this. Calling Office automation, IE legacy COM, or VBA-shaped APIs without `dynamic` requires a wall of `Type.InvokeMember` calls. With `[ComImport]` PIAs and `dynamic`, it reads like normal code.

  ```csharp
  dynamic excel = Activator.CreateInstance(Type.GetTypeFromProgID("Excel.Application")!)!;
  excel.Visible = true;
  dynamic book  = excel.Workbooks.Add();
  dynamic sheet = book.Worksheets[1];
  sheet.Cells[1, 1].Value = "Hello";
  ```

- **Dynamic-language hosting** (IronPython, IronRuby — historical now). The DLR was co-designed for this.
- **Wire-format scratchpads.** `dynamic d = JObject.Parse(json); d.user.id` reads cleanly — but in modern code, `JsonNode` (System.Text.Json) covers this without `dynamic`.
- **Multiple-dispatch / visitor with operator overloading.** `dynamic l + dynamic r` resolves the right `op_Addition`. Rare, but a real use case.
- **Test doubles.** Some BDD-style frameworks use `dynamic` to let you write `mock.Anything().Returns(42)`. NSubstitute and Moq don't need it; some hand-rolled doubles still do.

In greenfield application code on C# 14, the right answer is almost always not `dynamic`.

## `ExpandoObject` — a property bag with notify-on-change

`ExpandoObject` is the BCL's "I want to add properties at runtime" type. It implements `IDictionary<string, object>` and `INotifyPropertyChanged`:

```csharp
dynamic e = new ExpandoObject();
e.Name = "Ada";
e.Age  = 36;

// As a dictionary:
var dict = (IDictionary<string, object?>)e;
dict["Email"] = "ada@example.com";

((INotifyPropertyChanged)e).PropertyChanged += (_, ev) =>
    Console.WriteLine($"Changed: {ev.PropertyName}");
```

Useful for binding to XAML/Razor when the shape isn't known up front. Otherwise, prefer a real type or a `Dictionary<string, object>`.

## `DynamicObject` — bring your own dispatch

When you want `obj.AnyName` or `obj.AnyMethod(...)` to be intercepted, derive from `DynamicObject` and override the `Try*` virtuals:

```csharp
public sealed class FluentLogger : DynamicObject
{
    private readonly List<string> _calls = new();

    public override bool TryInvokeMember(
        InvokeMemberBinder binder, object?[]? args, out object? result)
    {
        _calls.Add($"{binder.Name}({string.Join(",", args ?? [])})");
        result = this;     // enable chaining
        return true;
    }

    public override string ToString() => string.Join(" → ", _calls);
}

dynamic log = new FluentLogger();
log.Step("a").Step("b").Step("c");
Console.WriteLine(log);   // Step(a) → Step(b) → Step(c)
```

The full set: `TryGetMember`, `TrySetMember`, `TryInvokeMember`, `TryConvert`, `TryBinaryOperation`, `TryUnaryOperation`, `TryGetIndex`, `TrySetIndex`, `TryDeleteMember`. Return `false` to fall back to default behaviour (which usually means `RuntimeBinderException`).

For deeper integration, implement `IDynamicMetaObjectProvider` directly — what `ExpandoObject` and `DynamicObject` both do internally — but that path is for language-runtime authors, not application code.

## Caveats that bite

- **No extension methods.** `dynamic` resolves against instance members only. `someDynamic.Where(x => …)` does *not* find `Enumerable.Where` — at runtime it looks for an instance method `Where` on the actual object. This is the single most common surprise.
- **No method-group conversions.** You can't pass a `dynamic`-typed expression where a delegate is expected without resolving it first.
- **Exceptions surface late.** Mistyping a member name compiles cleanly and throws `RuntimeBinderException` only when that line executes. There are no IDE squigglies, refactoring tools won't rename through `dynamic`, and code-coverage gaps become correctness gaps.
- **`async` interaction.** You can't `await` a `dynamic` directly without help — `await (dynamic)x` is allowed since C# 5, but the compiler has to work hard, and the result is `dynamic`, infecting the rest of the method.
- **Nullable reference types ignore `dynamic`.** A `dynamic` value is treated as not-null by the NRT analyzer; you don't get null warnings, you get NREs.
- **AOT incompatibility.** The DLR depends on `Reflection.Emit` to compile callsites. Under NativeAOT, the binder throws at first use. Set `<PublishAot>true</PublishAot>` and any `dynamic` usage will warn at build (analyser `IL3050` on the relevant code path).

## Performance — first call vs steady state

```
direct call            ~1 ns
warm CallSite (mono)   ~5-15 ns
warm CallSite (poly)   ~30-100 ns
cold CallSite          ~50-500 µs   (compiles a delegate the first time)
MethodInfo.Invoke      ~150-500 ns
```

Polymorphic callsites — same line, different runtime types each call — degrade fast as the inline cache fills (the L2 cache holds ~10 entries before falling back to the binder). If the same `dynamic d.Foo()` line sees a hot mix of types, you're back to per-call binder work. Reflection with a manual `Type → delegate` cache will outperform it.

## What to reach for instead in modern C#

| Situation                          | Modern alternative                                              |
|------------------------------------|-----------------------------------------------------------------|
| Working with arbitrary JSON        | `JsonNode` / `JsonElement` (System.Text.Json)                   |
| "Property bag" data flow           | `Dictionary<string, object>` or a record with `[JsonExtensionData]` |
| Duck-typed dispatch                | Generics + interface, or pattern matching (`is { Foo: var f }`) |
| Visitor over heterogeneous types   | Pattern matching with `switch` expression                       |
| COM interop                        | Still `dynamic` — that's what it's for                          |
| Plugin call by name                | Reflection + delegate cache, or source-generated dispatcher     |

## Senior-level gotchas

- **`dynamic` is viral.** Any expression involving a `dynamic` operand becomes `dynamic` (e.g. `int + dynamic` is `dynamic`), and the result silently propagates through your code until you cast it. Cast at the boundary: `int n = (int)d.X;`.
- **Extension methods don't exist for `dynamic`.** This includes LINQ. `dynamic xs = …; xs.Where(...)` will throw `RuntimeBinderException` at runtime. Cast to `IEnumerable<T>` first.
- **Expression trees can't contain `dynamic`.** EF Core, MongoDB, and any LINQ-to-something will refuse to translate a `dynamic`-typed lambda body.
- **`null` propagation.** `dynamic d = null; d.Foo()` throws `RuntimeBinderException` (not `NullReferenceException`) because the binder fails on the null target. Subtle when reading stack traces.
- **Boxing of value types is pervasive.** A `dynamic int` is conceptually `object` — every arithmetic op on `dynamic` value types involves a box and an unbox, even after the callsite is warm.
- **`dynamic` and overload resolution differ from compile-time C#.** Runtime overload resolution looks at the **runtime** types of the arguments, not the static types. `void F(object)` and `void F(string)`: at compile time, `string s` picks `F(string)`; with `dynamic d = "x"; F(d);`, runtime picks `F(string)` too — but if you have access modifiers or `params` mixed in, you can get genuinely different behaviour from the same source.
- **Don't store `dynamic` in fields/properties on long-lived objects.** Each new method that uses the field creates a new callsite cache; they live as long as the type does. Heavy `dynamic` use leaks small amounts of code-cache memory permanently.
- **`is` and `as` work, but `is dynamic` is not what you think.** `x is dynamic` is `x is object` (always true for non-null), because `dynamic` is `object` at runtime. Pattern matching on `dynamic` is unreliable; cast first.
- **Compiler emits a `Microsoft.CSharp.dll` reference.** A library that uses `dynamic` carries a transitive dependency on the C# binder; trim it out and the program crashes. The trim analyser will warn — heed it.
- **Cross-AppDomain / cross-`AssemblyLoadContext` `dynamic` is a minefield.** The cached callsite delegate is bound to the originating context; sending a `dynamic` across boundaries can pin both contexts in memory.
