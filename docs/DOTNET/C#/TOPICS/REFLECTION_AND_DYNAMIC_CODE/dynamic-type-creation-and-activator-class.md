# Dynamic Type Creation and the `Activator` Class

_Targets .NET 10 / C# 14. See also: [Reflection and member discovery](./reflection-and-member-discovery.md), [Dynamic keyword](./dynamic-keyword.md), [Interpolated string handlers](../GENERAL_TOPICS/interpolated-string-handlers.md)._

"Dynamic type creation" actually covers four distinct mechanisms with very different cost and capability:

| Tool                        | What it builds                            | Speed (relative)             | AOT-safe?               |
|-----------------------------|-------------------------------------------|------------------------------|-------------------------|
| `Activator.CreateInstance`  | An instance of a known `Type`             | ~100× slower than `new`      | Needs DAM annotations   |
| `Expression.Compile()`      | A delegate from an expression tree        | ~5–10× slower than `DynamicMethod` | Falls back to interpreter |
| `DynamicMethod`             | A method with raw IL, GC'd when unused    | Near-direct after JIT        | No (Reflection.Emit)    |
| `AssemblyBuilder` / `TypeBuilder` | Whole new types, possibly persisted | Build cost up front, runtime is normal | No                  |

Pick by what you actually need. Need an instance? `Activator`. Need a method? Expression tree first, drop to `DynamicMethod` if you've measured. Need a whole type (proxies, interceptors)? `Reflection.Emit` — and seriously consider whether a source generator does the same job at compile time.

## `Activator.CreateInstance` — the easy path

```csharp
// Parameterless ctor, generic.
var u = Activator.CreateInstance<User>();

// Parameterless ctor, by Type.
object u2 = Activator.CreateInstance(typeof(User))!;

// With ctor args (params object?[]).
var u3 = (User)Activator.CreateInstance(typeof(User), "Ada", 36)!;

// With BindingFlags + Binder for non-public ctors.
var u4 = (User)Activator.CreateInstance(
    typeof(User),
    BindingFlags.Instance | BindingFlags.NonPublic,
    binder: null,
    args:   ["Ada", 36],
    culture: null)!;
```

What it costs:

- **Constructor lookup** every call (`Type.GetConstructor` walks members; ambiguity rules apply).
- **`object?[]` argument array** allocates and forces value-type args to box.
- **Coercion** (binder picks a ctor, may convert `int` to `long`, etc.).

For a single one-shot construction, fine. For anything in a hot path (DI containers, deserialisers, mappers), measure and cache.

### Faster construction, in increasing complexity

```csharp
// 1. Cache a Func via Expression — ~50× faster than Activator on the steady state.
private static readonly Func<User> _newUser =
    Expression.Lambda<Func<User>>(Expression.New(typeof(User))).Compile();

// 2. With args:
private static readonly Func<string, int, User> _newUserWith =
    BuildCtor<Func<string, int, User>>(typeof(User), [typeof(string), typeof(int)]);

private static TDelegate BuildCtor<TDelegate>(Type t, Type[] argTypes) where TDelegate : Delegate
{
    var ctor = t.GetConstructor(argTypes)
               ?? throw new InvalidOperationException($"No ctor on {t}");
    var ps   = argTypes.Select(Expression.Parameter).ToArray();
    return Expression.Lambda<TDelegate>(Expression.New(ctor, ps), ps).Compile();
}
```

This is what every modern DI container, serialiser, and ORM does internally — build once at first-use, cache by `Type` (or `(Type, ConstructorInfo)`), invoke the delegate forever after.

## `Expression.Compile()` — runtime codegen with training wheels

Expression trees model code as data. You compose `Expression` nodes into a tree, call `Compile()`, and get back a delegate the JIT will optimise normally.

```csharp
// Build:  (x, y) => x.Total + y
ParameterExpression x = Expression.Parameter(typeof(Order), "x");
ParameterExpression y = Expression.Parameter(typeof(decimal), "y");

Expression body = Expression.Add(
    Expression.Property(x, nameof(Order.Total)),
    y);

Func<Order, decimal, decimal> f =
    Expression.Lambda<Func<Order, decimal, decimal>>(body, x, y).Compile();

decimal r = f(order, 10m);
```

Where it's used in production .NET:

- **EF Core** translates LINQ expression trees to SQL.
- **AutoMapper** compiles property-copy maps once per type pair.
- **STJ source generator's predecessor** (and still the runtime fallback) compiled property accessors via expressions.
- **Moq / NSubstitute** parse `mock.Setup(m => m.X)` lambdas as expression trees.

Cost model:

- `Compile()` is **expensive**: it walks the tree, lowers to IL, and JITs. Microseconds to milliseconds depending on tree size.
- After `Compile()`, the resulting delegate is essentially a regular method — full JIT optimisation, inlining where the call site permits.
- Each `Compile()` allocates IL into the `DynamicMethod` heap (collectible) — frequent recompiles can pressure the LOH-adjacent code-heap; cache the delegate, not the expression.

```csharp
// AOT-friendly variant:
var f = Expression.Lambda<...>(body, x, y).Compile(preferInterpretation: true);
```

The `preferInterpretation: true` overload skips Reflection.Emit and runs the tree through the BCL's expression interpreter — slower but works under NativeAOT and on iOS.

## `DynamicMethod` — when `Expression` is too slow

`DynamicMethod` lets you emit IL directly into a method object that's GC'd when unreferenced. No assembly bloat, no metadata pollution.

```csharp
// Equivalent of: int Get(Order o) => o._lines.Count;
FieldInfo lines = typeof(Order).GetField("_lines",
    BindingFlags.NonPublic | BindingFlags.Instance)!;

var dm = new DynamicMethod(
    name:        "Get_lines_count",
    returnType:  typeof(int),
    parameterTypes: [typeof(Order)],
    owner:       typeof(Order),
    skipVisibility: true);    // bypass private-field access checks

ILGenerator il = dm.GetILGenerator();
il.Emit(OpCodes.Ldarg_0);                        // o
il.Emit(OpCodes.Ldfld, lines);                   // o._lines
il.Emit(OpCodes.Callvirt,
    typeof(List<Line>).GetProperty("Count")!.GetMethod!);
il.Emit(OpCodes.Ret);

var getCount = dm.CreateDelegate<Func<Order, int>>();
int n = getCount(order);
```

Costs and constraints:

- IL emission is fast (microseconds).
- The resulting delegate runs at near-direct-call speed once JITted.
- The `DynamicMethod` and its delegate are collectible — drop all references and they go.
- `skipVisibility: true` requires the host to grant the assembly the right to bypass JIT visibility checks. In partial-trust scenarios it's denied; in modern .NET it's effectively allowed for in-process code, but the option still exists.

`DynamicMethod` is what underpins Castle DynamicProxy's interceptors, Sigil/Lokad-emit-style libraries, and the most performance-sensitive serialisers.

## `Reflection.Emit` — building whole types and assemblies

`AssemblyBuilder` → `ModuleBuilder` → `TypeBuilder` → method/property/field builders → `CreateType()`.

```csharp
AssemblyName name = new("Generated.Proxies");
AssemblyBuilder asm = AssemblyBuilder.DefineDynamicAssembly(
    name, AssemblyBuilderAccess.RunAndCollect);   // collectible at runtime

ModuleBuilder mod = asm.DefineDynamicModule("Main");
TypeBuilder tb = mod.DefineType("Generated.HelloProxy",
    TypeAttributes.Public | TypeAttributes.Class,
    parent: typeof(object),
    interfaces: [typeof(IGreeter)]);

MethodBuilder mb = tb.DefineMethod(nameof(IGreeter.Greet),
    MethodAttributes.Public | MethodAttributes.Virtual,
    returnType: typeof(string),
    parameterTypes: [typeof(string)]);

ILGenerator il = mb.GetILGenerator();
il.Emit(OpCodes.Ldstr, "Hello, ");
il.Emit(OpCodes.Ldarg_1);
il.Emit(OpCodes.Call, typeof(string).GetMethod(nameof(string.Concat),
    [typeof(string), typeof(string)])!);
il.Emit(OpCodes.Ret);

tb.DefineMethodOverride(mb, typeof(IGreeter).GetMethod(nameof(IGreeter.Greet))!);

Type proxy = tb.CreateType();
IGreeter g = (IGreeter)Activator.CreateInstance(proxy)!;
Console.WriteLine(g.Greet("Ada"));
```

`AssemblyBuilderAccess` options worth knowing:

- **`Run`** — the original; not collectible. Pinned for the process lifetime.
- **`RunAndCollect`** — collectible. The right default for most runtime-emit scenarios; the assembly and all its types/instances vanish when nothing roots them.

The legacy "save to disk" path (`AssemblyBuilderAccess.Save`) was removed in .NET Core. **`PersistedAssemblyBuilder`** (.NET 9+, in `System.Reflection.Emit` extensions) brings persistence back when you genuinely need to write a `.dll` from a running program — IL-rewriting tools, ahead-of-time compilation drivers, fuzzers.

## Source generators — the compile-time alternative

The historical reason to reach for `Reflection.Emit` was "I want to call a member that doesn't exist yet". In .NET 6+, that need is increasingly served at **compile time** by:

- **Source generators** (Roslyn `IIncrementalGenerator`) — see `[GeneratedRegex]`, `[JsonSerializable]`, `[LoggerMessage]`, EF Core 8+ compiled models.
- **Interceptors** (preview in C# 12, stabilising in C# 14) — replace specific call sites at build time.
- **Static abstract interface members** (C# 11) — generic dispatch over operators and statics without virtuals.

Compared to `Reflection.Emit`:

- **Debuggable** — you can step through generated source.
- **AOT-clean** — no `IL3050`, no warnings.
- **Reviewable** — generated files appear in the IDE and source control if you want.
- **Cold start** — zero runtime emit cost.

Reach for `Reflection.Emit` only when the type is genuinely unknown until runtime: dynamic-proxy frameworks (Castle), mocking libraries, scripting hosts, runtime-loaded plugins that need code rewriting. For everything else in greenfield code, source generators are the correct answer.

## AOT and trimming summary

| Mechanism                      | Under NativeAOT                                                    |
|--------------------------------|--------------------------------------------------------------------|
| `Activator.CreateInstance(Type)` | Works *if* the `Type` parameter has `[DynamicallyAccessedMembers(PublicConstructors)]` flowed to it; otherwise IL2070. |
| `Activator.CreateInstance<T>()`  | Works without annotations — the JIT sees the concrete `T`.       |
| `Expression.Compile()`           | Throws unless `preferInterpretation: true`.                      |
| `Expression.Compile(preferInterpretation: true)` | Works, runs the interpreter — ~5–20× slower than the JIT path. |
| `DynamicMethod`                  | **Not supported.** Throws `PlatformNotSupportedException`.        |
| `AssemblyBuilder` / `TypeBuilder`| **Not supported.** Throws.                                        |

If you're targeting NativeAOT, the codegen story is: source generators for shape-known-at-compile-time, `Activator.CreateInstance<T>()` or DAM-annotated reflection for runtime construction, expression interpreter for runtime expressions. `Reflection.Emit` is off the table.

## Senior-level gotchas

- **`Activator.CreateInstance(typeof(SomeStruct))` returns a boxed value.** Even though structs have an implicit parameterless ctor, the call boxes through `object`. For zero-alloc default value-type construction, use `default(T)` or `RuntimeHelpers.GetUninitializedObject` (which still boxes — there's no non-boxing reflection path).
- **`Activator.CreateInstance(typeof(T), null)` ≠ `(T)`.** The second-overload `args` of `null` is *no args* historically, but in modern C# with collection expressions it's easy to confuse with "one null arg". Be explicit: `args: []` or `args: [null]`.
- **`DynamicMethod.skipVisibility: true` is a real capability boundary.** It lets you read private fields across assembly lines. In tightly sandboxed hosts (game engines, plugins with capability-based security), the host may refuse. Document the requirement.
- **`Expression.Compile()` doesn't see closures the way C# does.** Capturing a local in an `Expression.Lambda` body requires modelling the capture explicitly via `Expression.Constant` or a `MemberExpression` over a closure object. The compiler does this for you when you write `Expression<Func<T>> e = () => x.Foo;`; if you're hand-building trees, you have to recreate it.
- **`AssemblyBuilderAccess.Run` is pinned forever.** Use `RunAndCollect` unless you're certain the assembly should live for the process lifetime. A long-running service that emits per-request runs out of code heap fast.
- **Generic methods built via `MethodBuilder.MakeGenericMethod` over `TypeBuilder`-built parameters** can deadlock the type loader during `CreateType` — emit ordering matters. Build dependencies first.
- **`ConstructorInfo.Invoke` vs `Activator.CreateInstance`** — both exist; `Activator` resolves the constructor for you, `Invoke` requires you to have the `ConstructorInfo` already. The former is convenient; the latter, when cached, is fractionally faster on subsequent calls.
- **Generated types' debuggability.** Without a `.pdb`, exceptions thrown in emitted IL show up with no source mapping. For `Reflection.Emit`, attach a `Debug` `ISymbolDocumentWriter` (or use `PersistedAssemblyBuilder` to write a `.dll` + `.pdb` and reload). For `Expression.Compile()`, use `DebugInfoGenerator.CreatePdbGenerator()`.
- **Pinning `Type` references in caches.** A `ConcurrentDictionary<Type, Func<object>>` will pin every type it sees forever — fine for application types, deadly if you're working with collectible `AssemblyLoadContext`s. Use `ConditionalWeakTable<Type, ...>` when the keys may be unloaded.
- **`RuntimeHelpers.CreateUninitializedObject`** bypasses ctors entirely — the value-type or class is zeroed, no field initializers run, no ctor side effects. Serialisers (formerly BinaryFormatter, now MessagePack and friends) use this. Don't reach for it without understanding that invariants the type relies on may be violated.
