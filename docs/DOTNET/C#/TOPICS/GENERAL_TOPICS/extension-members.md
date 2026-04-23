# Extension members

_Targets .NET 10 / C# 14. See also: [Default interface members](./default-interface-members.md), [Primary constructors](./primary-constructors.md), [Classes, records, structs](../OOP/classes-records-structs.md), [Generic constraints and variance](./generic-constraints-and-variance.md)._

Extension methods have existed since C# 3 — `static` methods in a `static` class whose first parameter carries `this`. C# 14 generalizes them into **extension members** with a proper `extension` block: instance and static methods, properties (instance and static), indexers, and operators, all authored inside one declaration over a receiver type. The old C# 3 extension method syntax still works; the new shape is strictly additive.

## The old shape (still valid)

```csharp
public static class StringExtensions
{
    public static bool IsNullOrEmpty(this string? s) => string.IsNullOrEmpty(s);
    public static string Truncate(this string s, int len) =>
        s.Length <= len ? s : s[..len];
}

"abcdef".Truncate(3);   // lowered to StringExtensions.Truncate("abcdef", 3)
```

At call sites the compiler lowers `receiver.Method(args)` into a static call `StaticClass.Method(receiver, args)`. The receiver type is resolved via attribute metadata (`[Extension]` on the method and containing class).

## The C# 14 shape — `extension` blocks

```csharp
public static class StringExtras
{
    extension(string s)                         // "for the receiver type 'string'"
    {
        public bool HasContent => !string.IsNullOrWhiteSpace(s);   // extension PROPERTY
        public string Truncate(int len) =>                         // extension METHOD
            s.Length <= len ? s : s[..len];
        public char this[Index i] => s[i.GetOffset(s.Length)];     // extension INDEXER
    }

    extension(string)                            // static-only receiver: extends the type
    {
        public static string EmptyJson => "{}";                    // static extension prop
        public static string Combine(params string[] parts) =>     // static extension method
            string.Concat(parts);
    }
}

// Usage:
"hello".HasContent;          // extension property
"hello".Truncate(3);         // extension method
"hello"[^1..];               // normal
string.EmptyJson;            // static extension property
string.Combine("a", "b");    // static extension method
```

Each `extension(receiver)` block groups members that share a receiver. The `(string s)` variant introduces a name you can use inside member bodies; `(string)` omits the name for static-only members. The compiler lowers every member to a regular static method with the receiver as the first parameter (the `[Extension]` attribute pattern is preserved for interop with older consumers).

## What's new vs. the old syntax

| Capability                               | Old (`this` parameter) | C# 14 `extension` blocks |
|------------------------------------------|------------------------|--------------------------|
| Extension methods                         | Yes                    | Yes                      |
| Extension properties                      | **No**                 | **Yes**                  |
| Extension indexers                        | No                     | **Yes**                  |
| Static extension methods (on the type)    | No                     | **Yes**                  |
| Static extension properties               | No                     | **Yes**                  |
| Extension operators (`op_Addition`, etc.) | No                     | **Yes** (C# 14)          |
| Generic type-parameter receivers          | Yes                    | Yes                      |
| Scoped by `using`                         | Yes                    | Yes                      |

Adoption of the new syntax is opt-in per-method; mixing styles inside one static class is legal.

## Resolution and ambiguity

Extension members are always **candidates last**. The compiler first looks for instance or static members declared on the type (or inherited). Only if lookup finds nothing does it widen to extension members in scope.

```csharp
public static class E
{
    extension(string s)
    {
        public int Length2 => s.Length * 2;      // no conflict — name is new
        public int Length  => 0;                 // hidden — string.Length always wins
    }
}

"abc".Length;    // 3  (instance wins)
"abc".Length2;   // 6  (extension)
```

Ambiguity between two extensions in scope produces CS0121. Narrow the `using` or fully-qualify:

```csharp
StringExtras.StringExtension(s).Truncate(3);  // synthesized helper name, rarely used
StringExtras.Truncate(s, 3);                   // classic static call still works
```

## Generic receivers and constraints

```csharp
public static class CollectionExtras
{
    extension<T>(IEnumerable<T> src) where T : notnull
    {
        public HashSet<T> ToHashSetOf() => new(src);
        public T First() => src.FirstOrDefault() ?? throw new InvalidOperationException();
    }

    extension<T>(IList<T> list) where T : struct
    {
        public void ShiftLeft() { /* mutate */ }
    }
}
```

Generic constraints behave exactly as on regular generic methods — including `where T : allows ref struct` for ref-struct-friendly extensions.

## Extension operators

```csharp
public static class VectorOps
{
    extension(Vector3 v)
    {
        public static Vector3 operator +(Vector3 a, Vector3 b) => new(a.X + b.X, a.Y + b.Y, a.Z + b.Z);
        public static Vector3 operator *(Vector3 a, float s)    => new(a.X * s, a.Y * s, a.Z * s);
    }
}
```

Rules:

- The declaring type must be a `static` class.
- At least one operand must match the receiver type (or a related generic parameter).
- Does **not** override or shadow user-defined operators on the receiver. Existing operator resolution wins.

This unblocks a long-standing pain point: you could never define operators over a type you didn't own. Now you can — though the call sites still bind to the existing operators first.

## Nullable receivers

```csharp
extension(string? s)
{
    public bool IsBlank => string.IsNullOrWhiteSpace(s);
}

string? x = null;
x.IsBlank;   // fine — receiver is nullable
```

A nullable receiver lets you call the extension on a possibly-null reference safely. The compiler does **not** null-check the receiver; the body must handle it.

## `using` scope and discoverability

Extension members are only visible when the containing `static` class's namespace is imported via `using`. A `global using` brings them everywhere; a scoped `using` confines them. Conventionally, put extensions in their target type's namespace (so `System.Text` for `string` extensions) — this matches IntelliSense expectations.

## Performance

- No runtime cost vs. a regular static method call — the compiler emits the same call.
- Extension **properties** lower to static method calls (there's no "property" concept for extensions in IL; the compiler generates a getter method and a `PropertyInfo`-shaped marker).
- Extension **indexers** likewise lower to `get_Item` / `set_Item` static methods.
- No allocation, no vtable indirection. Use them freely in hot paths.

## When extensions are the wrong tool

- **Where an interface fits better.** If you find yourself writing a dozen extensions that all perform the same shape of operation, that's an abstraction screaming to exist.
- **Surfacing hidden state.** Extensions cannot carry fields. Patterns like `[thread-static cache on extension]` reach for `ConditionalWeakTable` — usually a smell.
- **Replacing default interface members** for API evolution. Extensions cannot be overridden by implementers; DIM can.
- **On types you own.** If you can add the member directly to the type, do. Extensions are for types you don't own (BCL, third-party libraries) or cross-cutting helpers.

## Senior-level gotchas

- **Visibility lookup favors instance/static members on the receiver.** Adding a method with the same name to the receiver type in a future version silently takes precedence; your extension becomes dead code. Analyzer CS0618 / IDE0001 does not warn.
- **Extension methods don't participate in dynamic dispatch.** `dynamic d = "x"; d.Truncate(2);` throws — `dynamic` goes through the runtime binder which doesn't see `using`-imported extensions.
- **`this` parameter nullability is subtle.** With the old syntax, `static void M(this string s)` allowed `null` receivers silently; add `#nullable enable` and the compiler now flags it. `extension(string? s)` is the clean opt-in.
- **Extension indexers must be inside an `extension(type)` block.** You cannot add indexer extensions with the old `this` syntax.
- **Generic extension resolution uses type inference over the receiver first.** `new List<int>().Foo()` where `Foo` is `extension<T>(IEnumerable<T>)` infers `T=int`. If multiple extensions match, the more specific one wins (list > enumerable), and ties resolve as usual (CS0121 on ambiguity).
- **Extension operators do not change the `<` / `>` comparison operators for sorting.** `List<T>.Sort()` uses `IComparer<T>.Default`, not operators — an extension `operator<` on `MyType` does **not** make `List<MyType>.Sort()` work.
- **Call-site compatibility with old C# versions.** Consumers on C# ≤13 can call a method-shaped extension (lowered to a static method) but cannot access extension *properties*. Ship a lower language-version overload if you care.
- **`.ConfigureAwait(false)` is an extension method on the `TaskAwaiter` pattern** — not a special compiler feature. You can define your own awaitable extensions the same way.
- **Boxing of `struct` receivers.** `extension(IEnumerable<T> src)` called on a `List<T>` boxes the list's enumerator-struct when the extension enumerates through the interface. Prefer `extension<T>(List<T> list)` for hot loops.
- **Extension methods do not extend `ref struct`** unless you declare them with `allows ref struct` generics or take `ref ReadOnlySpan<T>` explicitly. Older BCL extensions (`MemoryExtensions`) use that pattern.
- **Operator extensions don't resolve across assemblies** unless the relevant `using` is in scope at the call site. A library shipping `operator +` extensions must document the `using`.
- **Don't define extensions on `object`.** Every lookup for every member name on every type becomes a candidate. IntelliSense pollution is unforgivable.
