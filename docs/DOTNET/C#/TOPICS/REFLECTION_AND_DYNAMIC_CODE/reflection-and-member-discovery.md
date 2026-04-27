# Reflection and Member Discovery

_Targets .NET 10 / C# 14. See also: [Dynamic keyword](./dynamic-keyword.md), [Dynamic type creation and Activator](./dynamic-type-creation-and-activator-class.md), [Tuples, Attributes, Regex](../GENERAL_TOPICS/tuples-attributes-regex.md), [Iterators and IAsyncDisposable](../GENERAL_TOPICS/iterators-and-iasyncdisposable.md)._

Reflection is the API surface that lets you read — and to a lesser extent, invoke — the metadata the compiler emits into every assembly. Every type, member, parameter, attribute, and generic argument is a row in a metadata table; `System.Reflection` is the dictionary you use to walk those tables at runtime.

It is also one of the most expensive things you can do in .NET. The senior question is rarely "how do I reflect over X" — it's "**should I**, and if yes, can I cache the result down to a delegate so I only pay the lookup once?".

## The metadata model

```
Assembly  ──→  Module  ──→  Type  ──→  Member (Method | Property | Field | Event | NestedType)
                                  └→  CustomAttributes
```

`Assembly` is loaded; `Module` is a sub-unit (almost always one per assembly); `Type` describes a class/struct/interface/enum/delegate; members hang off types. Everything in the BCL reflects (pun) this hierarchy.

```csharp
Assembly asm   = typeof(string).Assembly;        // System.Private.CoreLib
Module   mod   = asm.ManifestModule;
Type     str   = mod.GetType("System.String")!;
MethodInfo mi  = str.GetMethod("Substring", [typeof(int), typeof(int)])!;
```

## Getting a `Type`

| Source                       | When                                                             |
|------------------------------|------------------------------------------------------------------|
| `typeof(Foo)`                | You know the type at compile time. Compiles to `ldtoken` — cheap.|
| `obj.GetType()`              | You have an instance and want its **runtime** type.              |
| `Type.GetType("ns.Foo")`     | Loading by string from the **current or `mscorlib` assembly only** unless you provide an assembly-qualified name. |
| `assembly.GetType("ns.Foo")` | Looking up inside a specific assembly you already have.          |

```csharp
Type t1 = typeof(List<int>);
Type t2 = list.GetType();                                       // closed generic too
Type t3 = Type.GetType("MyApp.User, MyApp")!;                   // assembly-qualified
Type t4 = Assembly.LoadFrom("Plugin.dll").GetType("Plugin.X")!;
```

The trap with `Type.GetType("MyApp.User")` (no assembly) is that it returns **`null`** silently when the type lives outside the calling assembly. Use the assembly-qualified form, or hold an `Assembly` reference and call `GetType` on it, or pass `throwOnError: true` to fail loudly.

## `BindingFlags` — the default is a footgun

`Type.GetMembers()` defaults to `Public | Instance`. So:

```csharp
typeof(MyClass).GetMethod("DoIt");                  // public instance only
typeof(MyClass).GetMethod("DoIt",
    BindingFlags.NonPublic | BindingFlags.Instance);
typeof(MyClass).GetMethod("DoIt",
    BindingFlags.Public | BindingFlags.Static);
typeof(MyClass).GetMethods(
    BindingFlags.Public | BindingFlags.NonPublic |
    BindingFlags.Instance | BindingFlags.Static |
    BindingFlags.DeclaredOnly);                     // everything on this type, no inherited
```

The four combos worth memorising:

| Want                                       | Flags                                                           |
|--------------------------------------------|-----------------------------------------------------------------|
| All public + private instance              | `Public \| NonPublic \| Instance`                               |
| All static                                 | `Public \| NonPublic \| Static`                                 |
| This type only, ignore base                | add `DeclaredOnly`                                              |
| Everything                                 | `Public \| NonPublic \| Instance \| Static \| DeclaredOnly`     |

Without `DeclaredOnly`, `GetMethods` walks up to `object` and returns every inherited method — `ToString`, `Equals`, `GetHashCode` — which is rarely what you want when you're scanning for, say, `[Handler]`-attributed methods.

## Discovering members

```csharp
Type t = typeof(Order);

MethodInfo[]   methods    = t.GetMethods(BindingFlags.Public | BindingFlags.Instance);
PropertyInfo[] props      = t.GetProperties();
FieldInfo[]    fields     = t.GetFields(BindingFlags.NonPublic | BindingFlags.Instance);
EventInfo[]    events     = t.GetEvents();
ConstructorInfo[] ctors   = t.GetConstructors();
MemberInfo[]   everything = t.GetMembers();
```

`GetMethod(name)` throws `AmbiguousMatchException` if there are overloads. Disambiguate with the parameter-type array:

```csharp
var sub = typeof(string).GetMethod("Substring", [typeof(int), typeof(int)])!;
```

For interfaces, `GetMethods()` only returns the interface's own methods. To find what implements an interface method on a type, use `GetInterfaceMap`:

```csharp
InterfaceMapping map = typeof(MyHandler).GetInterfaceMap(typeof(IHandler));
// map.TargetMethods[i] is the concrete method that satisfies map.InterfaceMethods[i]
```

## Generics

A generic type definition (`List<>`) and a closed generic (`List<int>`) are different `Type` objects. Reflection lets you move between them:

```csharp
Type open    = typeof(List<>);                      // List`1, IsGenericTypeDefinition == true
Type closed  = open.MakeGenericType(typeof(int));   // List<int>
Type[] args  = closed.GetGenericArguments();        // [ typeof(int) ]
Type defAgain = closed.GetGenericTypeDefinition();  // List<>

bool isList = typeof(List<int>).IsGenericType
              && typeof(List<int>).GetGenericTypeDefinition() == typeof(List<>);
```

For methods, the same idea via `MakeGenericMethod`. You **cannot** `Invoke` an open generic method:

```csharp
MethodInfo def    = typeof(Foo).GetMethod(nameof(Foo.Cast))!;     // T Cast<T>(object)
MethodInfo closed = def.MakeGenericMethod(typeof(int));
int x = (int)closed.Invoke(null, [boxed])!;
```

## Custom attributes

```csharp
var route = typeof(UsersController).GetCustomAttribute<RouteAttribute>();
bool hasIt = typeof(UsersController).IsDefined(typeof(AuthorizeAttribute), inherit: true);

foreach (var attr in typeof(MyHandler).GetCustomAttributes<HandlerAttribute>())
    Console.WriteLine(attr.Pattern);
```

Three things to remember:

- `inherit: true` walks the base-class chain *for class-targeting attributes only*. It does not walk interfaces — `[ServiceLifetime]` on an interface won't be found on the implementing class via `inherit`.
- `IsDefined` is faster than `GetCustomAttribute` when you only need a yes/no — it doesn't materialise the attribute instance.
- Generic attributes (`[MyAttr<int>]`) are supported since C# 11; they reflect the same way.

See the attributes deep-dive in [tuples-attributes-regex.md](../GENERAL_TOPICS/tuples-attributes-regex.md).

## Invoking members

```csharp
MethodInfo   mi = typeof(MyService).GetMethod("Run")!;
object?      result = mi.Invoke(svc, [arg1, arg2]);

PropertyInfo pi = typeof(Order).GetProperty("Total")!;
decimal      total = (decimal)pi.GetValue(order)!;
pi.SetValue(order, 99m);

FieldInfo    fi = typeof(Order).GetField("_lines",
                    BindingFlags.NonPublic | BindingFlags.Instance)!;
var lines = (List<Line>)fi.GetValue(order)!;
```

Costs:

- Every `object[]` argument array allocates.
- Value-type arguments and return values **box** through `object`.
- The runtime walks visibility, security, and parameter-coercion rules on every call.

`MethodInfo.Invoke` is roughly **100×–1000× slower than a direct call** in tight loops. If you're calling more than a few times, build a delegate once and cache it.

## From `MethodInfo` to delegate (the only way to make reflection fast)

```csharp
// Open delegate: first arg is the receiver; works for instance methods.
var run = (Func<MyService, int, string>)Delegate.CreateDelegate(
    typeof(Func<MyService, int, string>), mi);

string s = run(svc, 42);   // ~ direct-call speed after JIT
```

For property getters/setters, `PropertyInfo.GetMethod` / `SetMethod` give you the underlying `MethodInfo` you can wrap as `Func<T, TProp>` / `Action<T, TProp>`.

When the shape isn't a known delegate (e.g. you only know the signature at runtime), drop down to `Expression.Compile()` or `DynamicMethod` — see [dynamic-type-creation-and-activator-class.md](./dynamic-type-creation-and-activator-class.md).

## Caching is not optional

```csharp
private static readonly ConcurrentDictionary<Type, Func<object>> _ctorCache = new();

public static object CreateDefault(Type t) =>
    _ctorCache.GetOrAdd(t, static type =>
        Expression.Lambda<Func<object>>(
            Expression.Convert(Expression.New(type), typeof(object))
        ).Compile())();
```

Every framework that "looks fast despite using reflection" — ASP.NET Core model binding, EF Core change tracking, AutoMapper, MediatR, System.Text.Json — does this. Reflect once at startup or first-use, build a delegate, cache by `Type` (or `(Type, MemberInfo)` key), invoke the delegate forever after.

## Trimming and NativeAOT

NativeAOT and IL trimming **statically analyse** which types and members are reachable. Reflection is opaque to that analysis — `typeof(T).GetMethod("DoIt")` is just a string from the linker's point of view, so the linker may strip the method or the whole type.

The contract is `[DynamicallyAccessedMembers]` on parameters/return values/fields:

```csharp
public static T Create<[DynamicallyAccessedMembers(
            DynamicallyAccessedMemberTypes.PublicConstructors)] T>()
    => (T)Activator.CreateInstance(typeof(T))!;
```

Methods that *can't* be made trim-safe annotate themselves with `[RequiresUnreferencedCode]`, which propagates the warning to callers. The trim/AOT analysers raise:

- **IL2026** — call to a `[RequiresUnreferencedCode]` method.
- **IL2070/IL2075/IL2090** — passing a `Type` without sufficient `[DynamicallyAccessedMembers]` to a reflective API.
- **IL3050** — reflection that's incompatible with AOT (`MakeGenericType` with a value-type arg the analyser couldn't see, `Reflection.Emit`, etc.).

If you're shipping NativeAOT, the answer is almost always: **replace the reflection with a source generator** that emits the same dispatch at compile time. System.Text.Json, `LoggerMessage`, regex, and EF Core 8+ all offer source-generator modes for exactly this reason.

## Senior-level gotchas

- **`GetMethod(name)` ambiguity** is the most common reflection bug — overloads silently throw. Always pass the parameter-type array unless you're certain there's only one method by that name.
- **`Type.GetType` only searches the calling assembly + corelib.** For plugin loaders, hold the `Assembly` and call `Assembly.GetType`.
- **Default `BindingFlags` is `Public | Instance`.** Private fields and statics return null without warning. Get burned once, then memorise the four combos above.
- **Open generics can't be invoked.** Forgetting `MakeGenericMethod` and just calling `Invoke` on the open `MethodInfo` throws `InvalidOperationException` at runtime — the C# compiler can't catch this.
- **Records' init-only setters are real setters.** `PropertyInfo.SetValue` works on `init` setters from reflection (the `init` access modifier is enforced only at the C# language level, via `modreq(IsExternalInit)`). This is how `System.Text.Json` deserialises records.
- **`GetCustomAttribute` allocates the attribute instance every call.** Cache the result, or use `IsDefined` for boolean checks.
- **`MethodInfo` from an interface vs implementation are different objects.** `typeof(IFoo).GetMethod("Bar") != typeof(Foo).GetMethod("Bar")` — for dispatch through `GetInterfaceMap`, you want the implementation side.
- **`Assembly.LoadFrom` vs `Assembly.LoadFile` vs `AssemblyLoadContext`.** The first two have subtle binding-context differences with a long history of "why does my plugin see two copies of the type?". For modern plugin hosts, use a custom `AssemblyLoadContext` and never `LoadFrom`.
- **`obj.GetType()` on a proxy** (Castle DynamicProxy, EF Core lazy-loading proxies) returns the **proxy** type, not the proxied type. Check the type's `BaseType` or use the proxy framework's helper; better, redesign so you don't need to ask.
- **Reflection bypasses nullability.** `PropertyInfo.SetValue(obj, null)` will happily store `null` into a `string` property declared non-nullable. NRT is a compile-time fiction; the runtime doesn't enforce it.
- **Don't reflect in hot paths under AOT.** Even when it works, the interpreter fallback can be 10× slower than the JIT path. Profile with `dotnet-counters` and watch the `System.Reflection` event source on .NET 9+.
