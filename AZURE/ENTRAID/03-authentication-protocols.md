# Authentication Protocols

> **Exam mapping:** AZ-204 *(authentication & authorization)* · AZ-305 *(identity design)*
> **One-liner:** OIDC + OAuth2 are the workhorses; SAML/WS-Fed for legacy SaaS; pick the right OAuth2 flow per app shape.
> **Related:** [app registrations](02-app-registrations-and-service-principals.md) · [managed identities](04-managed-identities.md) · [conditional access](06-conditional-access-and-mfa.md)

## The three token types

| Token | Format | Audience (`aud`) | Purpose | Lifetime (default) |
|-------|--------|------------------|---------|--------------------|
| **ID token** | JWT | The **client app** itself | Tells the client *who* signed in (OIDC). Never sent to APIs. | ~1 hour |
| **Access token** | JWT (or opaque for some 1P resources) | The **resource API** | Sent in `Authorization: Bearer …` to call APIs. | 60–90 min, randomized |
| **Refresh token** | Opaque | Identity platform | Exchanged for new access tokens. Long-lived (24h–90d depending on policy and device state). | Up to 90 days inactivity |

`scp` (delegated scopes, space-separated string) vs `roles` (app permissions or assigned app roles, array). API code checks one or the other based on token kind.

## v1 vs v2 endpoints

| | v1 | v2 |
|---|----|----|
| Endpoint | `https://login.microsoftonline.com/{tenant}/oauth2/...` | `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/...` |
| Resource targeting | `resource=https://graph.microsoft.com` | `scope=https://graph.microsoft.com/User.Read` |
| Personal MS accounts | No | Yes (`/common` or `/consumers`) |
| Token issuer (`iss`) | `sts.windows.net` | `login.microsoftonline.com/{tenantid}/v2.0` |
| New work | **Don't.** | **Use v2.** MSAL targets v2 by default. |

You can request `accessTokenAcceptedVersion: 2` in the manifest so your API issues v2-format access tokens (recommended).

## OAuth 2.0 flows — when to use which

| Flow | App shape | Secret? | Status |
|------|-----------|---------|--------|
| **Authorization Code + PKCE** | SPA, mobile, desktop, web app w/ user | No (PKCE) for public clients; secret/cert for confidential | **Recommended** for any user-facing app |
| **Client Credentials** | Daemon / service-to-service / cron | Yes (or FIC / MI) | **Recommended** for app-only |
| **On-Behalf-Of (OBO)** | Mid-tier API calls another API as the user | Confidential client | **Recommended** for tiered APIs |
| **Device Code** | TVs, CLIs, IoT (no browser) | No | OK — use sparingly |
| **Implicit** | (was for SPAs) | No | **Deprecated** — use auth code + PKCE |
| **ROPC** (Resource Owner Password Credentials) | Username+password from app | n/a | **Deprecated** — never on federated, never with MFA |
| **Hybrid** (`response_type=code id_token`) | Server-rendered web wanting both | Yes | Niche — OWIN/old ASP.NET MVC |

### Auth code + PKCE — the diagram

```
Client → /authorize?response_type=code&code_challenge=...        →  Entra
Entra  →  redirect_uri?code=AUTH_CODE                            ←  Browser
Client → /token  code + code_verifier + client_id (+ secret)     →  Entra
Entra  →  { access_token, id_token, refresh_token }              ←
```

PKCE protects against intercepted auth codes — required for public clients (SPA, mobile), recommended even for confidential clients.

### Client credentials — daemon flow

```
App → /token  grant_type=client_credentials
              client_id, client_secret_or_assertion
              scope=https://graph.microsoft.com/.default        →  Entra
Entra  →  { access_token }                                       ←
```

`/.default` means "give me everything that's been admin-consented for this app on this resource." You cannot ask for individual scopes in client credentials — it's all-or-nothing per resource.

### On-Behalf-Of

```
User → SPA → access_token (aud=API1)                  → API1
API1 → /token  grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
                assertion=<API1's incoming user token>
                requested_token_use=on_behalf_of
                scope=https://api2/.default            → Entra
Entra → { new access_token aud=API2 carrying user identity }
```

OBO requires the original token to be issued for API1 (audience), and API1 to have **delegated** permission to API2 with admin consent.

### Device code

```
CLI → /devicecode  →  { user_code, verification_uri, device_code }
User browses to verification_uri, types user_code, signs in.
CLI polls /token  grant_type=device_code  → access_token
```

## OIDC

OIDC = identity layer on top of OAuth 2.0. Adds `id_token`, the `openid` scope, and the discovery doc:

```
GET https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration
```

Returns `jwks_uri` (signing keys), endpoints, supported claims, etc. **Cache the JWKS keys** and rotate periodically — never trust an `id_token` without validating signature, `iss`, `aud`, `nbf`, `exp`, and `nonce`.

## SAML and WS-Fed

| | SAML 2.0 | WS-Federation |
|---|----------|---------------|
| Token format | Signed XML assertion | Same (SAML inside WS-Fed envelope) |
| Use cases | Most legacy SaaS (Salesforce, ServiceNow), gallery enterprise apps | ADFS-era .NET apps |
| Endpoints | `https://login.microsoftonline.com/{tenant}/saml2` | `.../wsfed` |
| Bindings | HTTP-POST or HTTP-Redirect | Passive (HTTP) |

For a SAML enterprise app, you configure: **Identifier (Entity ID)**, **Reply URL (ACS)**, **Sign-on URL**, and the **claim mappings** (NameID format, attributes). The signing certificate from Entra is uploaded into the SaaS app.

## MSAL

Microsoft Authentication Library — replaces ADAL (retired). Languages: .NET, Java, Python, JS, Node, Go (preview), iOS/macOS, Android.

```csharp
// .NET — confidential client (web app calling Graph)
var app = ConfidentialClientApplicationBuilder
    .Create(clientId)
    .WithClientSecret(secret) // or .WithCertificate(cert) or .WithClientAssertion(...)
    .WithAuthority(new Uri($"https://login.microsoftonline.com/{tenantId}"))
    .Build();

// Daemon: client credentials
var token = await app.AcquireTokenForClient(new[] { "https://graph.microsoft.com/.default" })
                     .ExecuteAsync();

// OBO: pass user assertion in
var oboToken = await app.AcquireTokenOnBehalfOf(
    new[] { "https://api2/.default" },
    new UserAssertion(incomingUserToken)).ExecuteAsync();
```

```python
# MSAL Python — auth code + PKCE for a desktop app
from msal import PublicClientApplication
app = PublicClientApplication(
    client_id, authority=f"https://login.microsoftonline.com/{tenant}")
result = app.acquire_token_interactive(scopes=["User.Read"])
print(result["access_token"])
```

> Use the **token cache** (in-memory for daemons, persisted for desktop). Pull tokens from cache; MSAL silently refreshes when needed.

## Continuous Access Evaluation (CAE)

Default access token lifetime is ~1 hour. With CAE, the resource (Graph, Exchange, SharePoint, others) re-validates the token in near-real-time on critical events:

| Event that revokes immediately |
|--------------------------------|
| User account deleted/disabled |
| Password change |
| MFA enabled |
| Conditional Access policy change |
| User detected as risky by Identity Protection |
| Token revocation by admin |

CAE-aware tokens have longer lifetime (~28h) but can be revoked instantly. Clients must be CAE-capable (claims challenge handling — return `WWW-Authenticate: Bearer error="insufficient_claims" claims=…` and reacquire). MSAL ≥ 4.43 supports CAE.

## Token validation checklist (what your API must do)

1. Validate signature against JWKS for the issuer.
2. Check `iss` matches your tenant's expected issuer (`v2.0`).
3. Check `aud` matches your API's app ID URI or client ID.
4. Check `exp`, `nbf`, allow ≤ 5 min skew.
5. Check `tid` (tenant) — for multi-tenant, against your allow-list.
6. For app-only tokens, check `roles` claim. For delegated, check `scp`.
7. Honor CAE claim challenges and reissue.

In ASP.NET Core, `Microsoft.Identity.Web` does most of this for you:

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));
```

## Exam traps

- **`/.default` is the only scope the client credentials flow accepts.** You can't request granular scopes there.
- **OBO requires admin consent on the downstream API** + delegated permission.
- **Device code flow needs `allowPublicClient: true`** in the manifest.
- **ID tokens are not for calling APIs.** Don't put one in `Authorization: Bearer`.
- **Implicit and ROPC are deprecated** — they remain in docs only as "do not use".
- **CAE means tokens can die mid-session.** Apps must handle 401 with `claims` challenge by re-acquiring.
- **Certificate-based auth needs the cert *and* a client assertion JWT** signed with it — MSAL builds that for you.
- **Bearer token over HTTPS only** — never log tokens; treat as passwords.

## Sources

- [Microsoft identity platform overview](https://learn.microsoft.com/en-us/entra/identity-platform/v2-overview)
- [MSAL authentication flows](https://learn.microsoft.com/en-us/entra/identity-platform/msal-authentication-flows)
- [Auth code + PKCE](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-auth-code-flow)
- [Client credentials](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow)
- [On-Behalf-Of](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow)
- [Implicit grant deprecation](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-implicit-grant-flow)
- [ROPC deprecation](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth-ropc)
- [Continuous Access Evaluation](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation)
