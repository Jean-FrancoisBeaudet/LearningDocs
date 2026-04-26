# SOAP (legacy)

_Targets .NET 10 / C# 14. See also: [REST](./rest.md), [gRPC](./grpc.md), [HttpClient & HttpClientFactory](./httpclient-and-httpclientfactory.md)._

SOAP is an XML-RPC protocol with a heavyweight envelope, a WSDL contract document, and a family of WS-\* extensions for security, reliability, and transactions. Modern .NET treats SOAP as **integration-only** — you build SOAP services or clients to talk to existing legacy systems (banks, BizTalk, government, hospital information systems, EDI gateways) and nothing else. The full WCF stack (`System.ServiceModel.*` server-side) was **dropped** in the move to .NET Core; today you have two viable paths: **CoreWCF** for hosting, and the `System.ServiceModel.*` client packages for consuming. If you control both ends, prefer REST or gRPC.

## Hosting: CoreWCF

CoreWCF is a community-led port of the WCF server-side surface to ASP.NET Core. It is **not** a Microsoft-supported product the same way ASP.NET Core is — it's open-source under the .NET Foundation, with Microsoft engineers contributing. Plan accordingly for support escalations.

```xml
<ItemGroup>
  <PackageReference Include="CoreWCF.Primitives" Version="1.7.*" />
  <PackageReference Include="CoreWCF.Http"       Version="1.7.*" />
  <PackageReference Include="CoreWCF.WebHttp"    Version="1.7.*" />
</ItemGroup>
```

```csharp
using CoreWCF;
using CoreWCF.Configuration;
using CoreWCF.Description;

[ServiceContract(Namespace = "https://acme.com/orders")]
public interface IOrdersService
{
    [OperationContract]
    Task<Order?> GetAsync(long id);
}

public sealed class OrdersService(IOrderRepo repo) : IOrdersService
{
    public Task<Order?> GetAsync(long id) => repo.FindAsync(id, CancellationToken.None);
}

[DataContract(Namespace = "https://acme.com/orders")]
public sealed class Order
{
    [DataMember(Order = 0)] public long   Id   { get; set; }
    [DataMember(Order = 1)] public string Sku  { get; set; } = "";
}

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddServiceModelServices()
                .AddServiceModelMetadata();
builder.Services.AddSingleton<IOrdersService, OrdersService>();

var app = builder.Build();

app.UseServiceModel(b =>
{
    b.AddService<OrdersService>(o => o.DebugBehavior.IncludeExceptionDetailInFaults = builder.Environment.IsDevelopment())
     .AddServiceEndpoint<OrdersService, IOrdersService>(new BasicHttpBinding(), "/Orders.svc")
     .AddServiceEndpoint<OrdersService, IOrdersService>(new WSHttpBinding(), "/Orders.wsbinding");
});

// Expose WSDL at /Orders.svc?wsdl
var smb = app.Services.GetRequiredService<ServiceMetadataBehavior>();
smb.HttpGetEnabled = true;
smb.HttpsGetEnabled = true;

app.Run();
```

Bindings — pick the right one for the legacy peer:

| Binding | Wire | Security | Use |
|---|---|---|---|
| `BasicHttpBinding` | SOAP 1.1, HTTP | None by default | Java AXIS / older Apache CXF, public WSDL-first endpoints |
| `WSHttpBinding` | SOAP 1.2, HTTP | WS-Security, WS-ReliableMessaging | Enterprise interop |
| `NetTcpBinding` | Binary, TCP | Windows / certificate | Windows-to-Windows internal |
| `WebHttpBinding` | "REST-ish over WCF" | Per-binding | Legacy "SOAP server also exposes JSON" — avoid for new work |

## Consuming a SOAP service

The client surface stayed in .NET — the packages were renamed and split:

```xml
<PackageReference Include="System.ServiceModel.Primitives" Version="8.*" />
<PackageReference Include="System.ServiceModel.Http"       Version="8.*" />
<PackageReference Include="System.ServiceModel.NetTcp"     Version="8.*" />
<PackageReference Include="System.ServiceModel.Federation" Version="8.*" />
```

Generate the proxy from a WSDL with `dotnet-svcutil`:

```bash
dotnet tool install --global dotnet-svcutil
dotnet-svcutil https://partner.example.com/Orders.svc?wsdl --namespace "*,Acme.Partner.Orders" --outputDir Generated
```

That emits `Reference.cs` containing an `OrdersServiceClient` and the `[DataContract]` types.

```csharp
var binding = new BasicHttpBinding(BasicHttpSecurityMode.Transport);  // force TLS
var endpoint = new EndpointAddress("https://partner.example.com/Orders.svc");

await using var client = new OrdersServiceClient(binding, endpoint);
client.ClientCredentials.UserName.UserName = config["Partner:Username"];
client.ClientCredentials.UserName.Password = config["Partner:Password"];

var order = await client.GetAsync(42);
await client.CloseAsync();                   // CommunicationObject lifecycle — see gotchas
```

### WS-Security

For SAML / X.509 / username-token over message security, `System.ServiceModel.Federation` provides `WSFederationHttpBinding` and `WS2007FederationHttpBinding`. Configure issued-token parameters:

```csharp
var binding = new WS2007FederationHttpBinding(WSFederationHttpSecurityMode.TransportWithMessageCredential);
binding.Security.Message.IssuedKeyType = SecurityKeyType.SymmetricKey;
binding.Security.Message.EstablishSecurityContext = false;
binding.Security.Message.IssuerAddress = new("https://sts.partner.com/issue");
binding.Security.Message.IssuerBinding = new WS2007HttpBinding(SecurityMode.Transport);
```

WS-Security is where SOAP earns its weight — REST has no equivalent for "encrypt this single field with the recipient's public key, sign the rest with my key, and route through three hops."

### MTOM (binary attachments)

Large binaries shouldn't live as base64-in-XML — that's a 33% size penalty. MTOM (Message Transmission Optimization Mechanism) sends the binary as a MIME attachment alongside the SOAP envelope:

```csharp
var binding = new BasicHttpBinding { MessageEncoding = WSMessageEncoding.Mtom };
```

A `byte[]` or `Stream` field over MTOM is wire-efficient. Streamed transfer mode (`TransferMode.Streamed`) keeps it out of memory entirely:

```csharp
binding.TransferMode = TransferMode.Streamed;
binding.MaxReceivedMessageSize = long.MaxValue;
// Contracts that stream MUST have exactly one parameter typed Stream or Message.
```

## Message contracts vs data contracts

`[DataContract]` describes a typed payload — clean, but you can't control envelope-level details. When the legacy peer demands specific SOAP headers or wraps:

```csharp
[MessageContract(IsWrapped = true, WrapperName = "GetOrderRequest")]
public sealed class GetOrderRequest
{
    [MessageHeader(Name = "TenantId", Namespace = "https://acme.com/")]
    public string TenantId { get; set; } = "";

    [MessageBodyMember(Order = 0)]
    public long Id { get; set; }
}
```

Use this when interop demands particular envelope shapes; otherwise `[DataContract]` is plenty.

## Migration paths

When stuck inheriting a SOAP integration:

1. **Adapter / strangler façade**: stand up a REST or gRPC service in front of the SOAP one. New clients hit the modern API; the old SOAP backend stays untouched until retirement. The façade is the only thing speaking SOAP.
2. **Direct port**: rare — only viable if you control the consumers and can co-deploy. Replace `BasicHttpBinding` with REST + JSON, regenerate clients, kill the WSDL.
3. **Keep CoreWCF in place indefinitely**: budget for the maintenance risk of a community-led project on the critical path.

## Modern note (.NET 10)

- The classic `System.ServiceModel.*` server-side namespaces (`System.ServiceModel.Activation`, IIS `*.svc` handlers, `BasicHttpBinding` server, `Microsoft.Web.Services3`) are .NET Framework only. They will never come back. Anything new on .NET Core / .NET 5+ goes through CoreWCF.
- `dotnet-svcutil` replaced the Visual Studio "Add Service Reference" dialog for WSDL-first development. The VS dialog still exists but invokes the same tool under the hood.
- CoreWCF v1.7+ supports the standard ASP.NET Core hosting model — pipelines, DI, `ILogger<T>`, `IConfiguration` all work as usual. You can mix CoreWCF and Minimal APIs in a single host.
- `XmlSerializer`-backed contracts (`[XmlSerializerFormat]`) work for legacy interop where `DataContractSerializer`'s alphabetic ordering is wrong.

## Senior-level gotchas

- **`BasicHttpBinding` defaults are insecure**: no TLS, no auth, plaintext on the wire. Use `BasicHttpSecurityMode.Transport` minimum, `TransportWithMessageCredential` for username/password.
- **`ServiceClient` is not `IHttpClientFactory`-managed**. Each `new OrdersServiceClient(...)` opens its own channel. Repeatedly minting them under load reproduces the original `HttpClient` socket-exhaustion problem from a different direction. **Cache one client per endpoint** and treat it like a singleton.
- **`CommunicationObject` close-semantics**: a faulted channel cannot be `CloseAsync`'d — you must `Abort()`. The classic try/catch dance:
  ```csharp
  try   { await client.CloseAsync(); }
  catch { client.Abort(); throw; }
  ```
  `await using` calls `Dispose`, which on a faulted channel throws — masking the original fault. Wrap explicitly.
- **`DataContractSerializer` orders members alphabetically by default**. Set `[DataMember(Order = ...)]` on every member or interop with strict peers will fail with no useful error. The peer just sees an unexpected element name and rejects the envelope.
- **`Stream`-typed contracts force `TransferMode.Streamed`** and require exactly one `Stream` or `Message` parameter on the operation. Mixing streamed and buffered ops on the same binding is not supported — split bindings.
- **`IncludeExceptionDetailInFaults = true` leaks stack traces and internal types into the SOAP fault**. Dev only. In prod, surface domain-meaningful `FaultException<T>` instead.
- **WSDL caching**: clients regenerated from a refreshed WSDL get new generated types. If the legacy peer changes namespaces (even just adding a new field), you face a recompile of every consumer. Persist generated code in source control and treat regeneration as a deliberate version bump, not a build step.
- **No async streams**. SOAP/WCF predate `IAsyncEnumerable<T>`; large result sets either stream as `Stream` (binary) or arrive as one materialized array. Plan for memory.
- **Time zones**: `DateTime` over the wire serializes as ISO-8601 with offset; round-tripping `DateTimeKind.Local` ↔ `DateTimeKind.Utc` between heterogeneous stacks is the #1 source of "the system was working yesterday" bugs.
