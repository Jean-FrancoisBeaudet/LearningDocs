# Kafka Security — SASL, SSL/TLS, ACLs, OAuth 2.0, Delegation Tokens, Quotas

> Version baseline: Apache Kafka 4.x (KRaft mode). Last researched: 2026-04.

## TL;DR

Kafka security is composed of three orthogonal layers that must each be configured:

1. **Transport encryption** — SSL/TLS on every listener (`SASL_SSL` or `SSL`). Plaintext is never acceptable outside a lab.
2. **Authentication** — proves *who* the client is. Options: mTLS (client cert), SASL/PLAIN, SASL/SCRAM-SHA-256/512, SASL/GSSAPI (Kerberos), SASL/OAUTHBEARER (OIDC/JWT), delegation tokens.
3. **Authorization** — proves *what* the authenticated principal may do. Options: `StandardAuthorizer` (KRaft-native ACLs, replaces the old ZK-based `AclAuthorizer`), or Confluent RBAC (centralized role bindings in MDS).

Plus cross-cutting concerns:

- **Quotas** (byte-rate + request-rate) for noisy-neighbor protection, keyed on `user` and/or `client-id`.
- **Encryption at rest** — Kafka does not encrypt log segments itself; rely on dm-crypt / LUKS / EBS encryption / KMS-managed volumes.
- **Secrets distribution** — keystores, keytabs, OAuth client secrets, SCRAM credentials (now stored in the KRaft metadata log, not ZK).

The Kafka 4.x baseline is **KRaft-only** (ZooKeeper removed in 4.0), so SCRAM credentials, delegation tokens, ACLs, and quotas are all stored in the `__cluster_metadata` log and replicated via the Raft controller quorum. This changes day-2 operational workflows significantly (see section on KRaft ACL storage).

## Deep dive

### 1. Listeners and listener security protocols

A broker advertises one or more listeners, each with a `listener.security.protocol.map` entry:

| Protocol    | Encryption | AuthN                              |
|-------------|-----------|-------------------------------------|
| `PLAINTEXT` | none      | none                                |
| `SSL`       | TLS       | none, or mTLS if `ssl.client.auth=required` |
| `SASL_PLAINTEXT` | none | SASL                                |
| `SASL_SSL`  | TLS       | SASL (auth) + optional mTLS         |

Typical production layout:

```properties
listeners=INTERNAL://:9092,EXTERNAL://:9093,CONTROLLER://:9094
advertised.listeners=INTERNAL://broker-1.internal:9092,EXTERNAL://kafka.example.com:9093
listener.security.protocol.map=INTERNAL:SASL_SSL,EXTERNAL:SASL_SSL,CONTROLLER:SASL_SSL
inter.broker.listener.name=INTERNAL
sasl.enabled.mechanisms=SCRAM-SHA-512,OAUTHBEARER
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
sasl.mechanism.controller.protocol=SCRAM-SHA-512
```

Key senior-level points:
- **Inter-broker listener must always be authenticated**, even on a private VLAN — a compromised host inside the VLAN otherwise gets full fetch/replica access.
- **Controller listener (KRaft)** is a separate security surface. In 4.x this is required — there is no ZK fallback.
- **Per-listener KeyStore/TrustStore** is supported (`listener.name.external.ssl.keystore.location=…`). Use this to terminate client mTLS on the external listener but use a simpler internal CA on the inter-broker listener.

### 2. SASL mechanisms

#### 2.1 SASL/PLAIN
Username/password over TLS. Credentials are configured broker-side via `JaasConfig` or a `PlainLoginModule` callback. Fine for bootstrapping or service accounts if you must, but:
- Passwords stored in plaintext on the broker (JAAS file) unless you plug in a custom `AuthenticateCallbackHandler` backed by e.g. Vault.
- No rotation support out of the box.

#### 2.2 SASL/SCRAM-SHA-256 and SCRAM-SHA-512
Salted challenge-response — password never crosses the wire. Credentials live in Kafka itself. In 4.x they are written to the metadata log:

```bash
# Create or update a SCRAM user
bin/kafka-configs.sh --bootstrap-server broker:9092 \
  --alter --add-config 'SCRAM-SHA-512=[iterations=8192,password=s3cr3t]' \
  --entity-type users --entity-name svc-orders
```

Prefer **SHA-512**; SHA-256 exists only for clients that cannot do 512. Iterations ≥ 8192 (default 4096 is weak).

#### 2.3 SASL/GSSAPI (Kerberos)
Enterprise Hadoop/Active Directory shops. Requires keytabs on every broker and client, a working KDC, reverse-DNS that matches principals. Operational overhead is high — most greenfield 4.x deployments pick OAUTHBEARER or mTLS instead. Still common in regulated / on-prem environments.

#### 2.4 SASL/OAUTHBEARER (OIDC) — KIP-255 + KIP-768
- **KIP-255** introduced the SASL mechanism and pluggable callback handlers.
- **KIP-768** (Kafka 3.1+, production-grade in 4.x) ships a default OIDC-compliant `OAuthBearerLoginCallbackHandler` and `OAuthBearerValidatorCallbackHandler`. It speaks the OAuth 2.0 *client_credentials* grant against an IdP (Keycloak, Okta, Entra ID, Auth0) and validates JWTs against a JWKS endpoint.
- KIP-768 was further extended (KAFKA-15878) to support **opaque (non-JWT) tokens** via token-introspection endpoints.

Client-side config (snippet):

```properties
security.protocol=SASL_SSL
sasl.mechanism=OAUTHBEARER
sasl.login.callback.handler.class=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginCallbackHandler
sasl.oauthbearer.token.endpoint.url=https://idp.example.com/realms/kafka/protocol/openid-connect/token
sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required \
  clientId="svc-orders" clientSecret="…" scope="kafka";
```

Broker-side validation:

```properties
sasl.oauthbearer.jwks.endpoint.url=https://idp.example.com/realms/kafka/protocol/openid-connect/certs
sasl.oauthbearer.expected.audience=kafka
sasl.oauthbearer.expected.issuer=https://idp.example.com/realms/kafka
sasl.oauthbearer.sub.claim.name=preferred_username   # what becomes the Kafka principal
```

**Gotchas**:
- The *principal* the authorizer sees is whatever claim you map via `sub.claim.name`. If you get that wrong, ACLs silently don't match.
- Cache the JWKS — brokers hit the IdP every time the keys rotate; a flapping IdP becomes a broker outage.
- Token lifetime vs. long-lived consumers: the client re-authenticates before expiry (KIP-368, re-authentication for SASL connections, `connections.max.reauth.ms`). Without this, connections that outlive the token get a silent `SaslAuthenticationException`.

#### 2.5 Delegation tokens
- A short-lived shared secret issued by the broker to an already-authenticated principal.
- Use case: Spark/Flink executors, Kafka Connect workers, MirrorMaker tasks — workloads that shouldn't carry keytabs or long-lived OAuth credentials.
- The token *impersonates* the owner (with an optional set of renewers) and is authenticated via `SASL/SCRAM-SHA-256` under the hood.
- In KRaft 4.x, tokens are persisted in the metadata log with automatic expiration.

```bash
bin/kafka-delegation-tokens.sh --bootstrap-server broker:9092 \
  --command-config admin.properties --create --max-life-time-period -1 \
  --renewer-principal User:flink-worker
```

### 3. SSL/TLS and mTLS

Core flags:

```properties
ssl.keystore.type=PKCS12
ssl.keystore.location=/etc/kafka/ssl/broker.p12
ssl.keystore.password=…
ssl.key.password=…
ssl.truststore.type=PKCS12
ssl.truststore.location=/etc/kafka/ssl/truststore.p12
ssl.truststore.password=…
ssl.client.auth=required           # = mTLS
ssl.endpoint.identification.algorithm=HTTPS   # verifies SAN on client side
ssl.enabled.protocols=TLSv1.3,TLSv1.2
ssl.cipher.suites=TLS_AES_128_GCM_SHA256,TLS_AES_256_GCM_SHA384,…
```

Principal mapping for mTLS uses `ssl.principal.mapping.rules` to extract a usable name from the cert DN:

```properties
ssl.principal.mapping.rules=RULE:^CN=([^,]+),.*$/$1/,DEFAULT
```

Senior points:
- **Prefer TLSv1.3** — faster handshake, no `RSA_WITH_*` baggage.
- **mTLS is the lowest-operational-cost auth** for machine-to-machine, but cert *rotation* at scale requires automation (cert-manager, Vault PKI). Brokers pick up new keystores only on listener reconfiguration (dynamic), not arbitrary JVM restarts — see `kafka-configs --alter --add-config ssl.keystore.location=…`.
- **Do not terminate TLS at a load balancer** in front of Kafka — clients need the broker's identity (SAN) to match `advertised.listeners`. Use TCP pass-through.
- **Intermediate CA churn** — when your internal PKI rotates an intermediate, update the broker truststore first (rolling restart), then clients.

### 4. Authorization — ACLs

Principal → Resource → Operation → Permission (Allow/Deny) → Host.

Resource types in 4.x: `Cluster`, `Topic`, `Group`, `TransactionalId`, `DelegationToken`, `User` (for quotas).

Operations: `Read`, `Write`, `Create`, `Delete`, `Alter`, `Describe`, `ClusterAction`, `DescribeConfigs`, `AlterConfigs`, `IdempotentWrite`, `CreateTokens`, `DescribeTokens`, plus `All`.

```bash
# Allow the orders-service to produce to the orders topic
bin/kafka-acls.sh --bootstrap-server broker:9092 \
  --add --allow-principal User:CN=orders-svc \
  --operation Write --operation Describe \
  --topic orders --resource-pattern-type literal

# Prefix-based ACL — orders-service can consume any orders.* topic as group orders-app
bin/kafka-acls.sh --bootstrap-server broker:9092 \
  --add --allow-principal User:CN=orders-svc \
  --operation Read --operation Describe \
  --topic orders. --resource-pattern-type prefixed \
  --group orders-app
```

Authorizer class in 4.x KRaft: `org.apache.kafka.metadata.authorizer.StandardAuthorizer`. (The old `kafka.security.authorizer.AclAuthorizer` was ZK-backed and is gone.)

```properties
authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer
super.users=User:admin;User:CN=controller-quorum
allow.everyone.if.no.acl.found=false   # never set true in prod
```

Deny wins over Allow; explicit is better than implicit.

#### Idempotent / transactional producers
Enabling `enable.idempotence=true` (default in 3.x+) requires the principal to have `IdempotentWrite` on `Cluster` (4.x relaxed this in some KIPs — verify on your version). Transactional producers additionally need `Write`/`Describe` on the `TransactionalId` resource.

#### Prefix ACLs
Always prefer prefixed patterns for multi-tenant clusters. An env/team prefix convention (e.g. `prod.payments.orders`) plus prefix ACLs scales to hundreds of services; literal ACLs do not.

### 5. Confluent RBAC vs vanilla ACLs

Vanilla ACLs:
- Stored in the metadata log of the cluster they apply to.
- Principal is a bare string (`User:CN=foo`) — no notion of groups, roles, or hierarchy.
- Scales poorly past ~thousands of ACLs per cluster: `kafka-acls --list` becomes painful; duplication across environments is manual.

Confluent RBAC (Confluent Platform 6.0+ / Confluent Cloud):
- Centralized Metadata Service (MDS) holds role bindings.
- Predefined roles: `SystemAdmin`, `ClusterAdmin`, `UserAdmin`, `SecurityAdmin`, `Operator`, `ResourceOwner`, `DeveloperRead`, `DeveloperWrite`, `DeveloperManage`.
- Scopes are hierarchical: organization → environment → cluster → resource. A `DeveloperWrite` on `topic=orders.*` in `env=prod` cascades correctly.
- Integrates with LDAP / OIDC to expand group memberships server-side.
- Coexists with ACLs — RBAC role bindings produce ACLs under the hood for cross-component enforcement. From CP 7.8+, **mTLS-based RBAC** is supported (previously RBAC required a token-based flow).

Rule of thumb: under ~20 services, ACLs are fine. Over that, and especially with multiple clusters or environments, RBAC pays for itself.

### 6. Quotas

Two dimensions:

| Quota type     | Units                         | Use case                                    |
|----------------|-------------------------------|---------------------------------------------|
| Produce byte rate   | bytes/sec per broker    | Cap a client's inbound traffic               |
| Fetch byte rate     | bytes/sec per broker    | Cap a client's outbound traffic              |
| Request rate        | % of a single broker network+IO thread (window-based) | Cap expensive metadata/consumer-offset traffic |
| Controller mutation | rate of topic/partition/ACL creation requests | Stop a rogue admin from DoS-ing the controller |

Keyed by any combo of `user`, `client-id`, `user + client-id` (most specific wins).

```bash
# Cap svc-orders at 50 MiB/s produce, 100 MiB/s fetch
bin/kafka-configs.sh --bootstrap-server broker:9092 --alter \
  --add-config 'producer_byte_rate=52428800,consumer_byte_rate=104857600' \
  --entity-type users --entity-name svc-orders
```

Senior points:
- Quotas are **per-broker, not per-cluster**. A 50 MB/s quota on a 12-broker cluster permits 600 MB/s aggregate if the partition leadership is spread evenly.
- Throttling kicks in by **delaying responses**, not by dropping. A misbehaving client sees rising `request-latency-avg` rather than errors — be sure your producer's `delivery.timeout.ms` is larger than expected throttle delays or you'll get false `TimeoutException`s.
- Monitor `kafka.server:type=ClientQuotaManager,user=<u>,client-id=<c>,name=throttle-time`.

### 7. Encryption at rest

Kafka does not natively encrypt log segments. Options:
- **Volume-level encryption** — dm-crypt / LUKS on-prem, EBS encryption + KMS on AWS, Managed Disk encryption on Azure, PD CMEK on GCP. Simple, transparent, covers the snapshot/backup path.
- **Client-side encryption** — producer encrypts payloads (e.g. Confluent Schema Registry CSFLE, or roll your own with AES-GCM + a KMS-wrapped DEK). Kafka never sees plaintext. Trade-off: schema evolution, searchability, and ksqlDB break.
- **Topic-level encryption plugins** — Conduktor Gateway, AWS MSK's integrated KMS per-topic option — a proxy/interceptor encrypts on the way in and decrypts on the way out.

GDPR/"right to be forgotten" is awkward on an append-only log. Patterns: short retention + tombstones on a compacted "PII projection" topic; or crypto-shredding (rotate/destroy the per-subject DEK).

### 8. KRaft-specific security operations

- SCRAM user creation, ACL creation, quota creation and delegation-token issuance all flow through the **active controller** and are persisted in `__cluster_metadata`.
- `super.users` on the controllers must include the inter-controller principal or the quorum cannot bootstrap.
- Rolling a controller's keystore requires leadership transfer to avoid a quorum blip — use `kafka-metadata-quorum.sh --bootstrap-server … describe --status` before restart.

## Config reference

### Broker / KRaft security configs

| Property | Typical value | Notes |
|---|---|---|
| `listeners` | `INTERNAL://:9092,EXTERNAL://:9093,CONTROLLER://:9094` | Controller listener mandatory in KRaft |
| `listener.security.protocol.map` | `INTERNAL:SASL_SSL,EXTERNAL:SASL_SSL,CONTROLLER:SASL_SSL` | |
| `inter.broker.listener.name` | `INTERNAL` | |
| `sasl.enabled.mechanisms` | `SCRAM-SHA-512,OAUTHBEARER` | List per broker |
| `sasl.mechanism.inter.broker.protocol` | `SCRAM-SHA-512` | |
| `sasl.mechanism.controller.protocol` | `SCRAM-SHA-512` | KRaft-specific |
| `ssl.client.auth` | `required` | mTLS on |
| `ssl.enabled.protocols` | `TLSv1.3,TLSv1.2` | |
| `ssl.endpoint.identification.algorithm` | `HTTPS` | SAN-checked |
| `ssl.principal.mapping.rules` | `RULE:^CN=([^,]+).*$/$1/,DEFAULT` | DN → principal |
| `authorizer.class.name` | `org.apache.kafka.metadata.authorizer.StandardAuthorizer` | KRaft native |
| `allow.everyone.if.no.acl.found` | `false` | Never `true` in prod |
| `super.users` | `User:admin;User:CN=controller` | Semicolon-separated |
| `sasl.oauthbearer.jwks.endpoint.url` | `https://idp/.../certs` | |
| `sasl.oauthbearer.expected.audience` | `kafka` | |
| `sasl.oauthbearer.expected.issuer` | `https://idp/...` | |
| `sasl.oauthbearer.sub.claim.name` | `preferred_username` | Principal mapping |
| `connections.max.reauth.ms` | `3600000` | Forces re-auth for OAuth tokens |
| `delegation.token.secret.key` | random 32+ bytes | Required to enable tokens |
| `delegation.token.expiry.time.ms` | `86400000` | 24h default |
| `delegation.token.max.lifetime.ms` | `604800000` | 7 days |
| `quota.window.num` / `quota.window.size.seconds` | `11` / `1` | Rolling window for quotas |

## .NET / C# snippet (Confluent.Kafka)

### SASL_SSL with SCRAM-SHA-512

```csharp
var producerConfig = new ProducerConfig
{
    BootstrapServers = "broker-1:9093,broker-2:9093,broker-3:9093",
    SecurityProtocol = SecurityProtocol.SaslSsl,
    SaslMechanism    = SaslMechanism.ScramSha512,
    SaslUsername     = "svc-orders",
    SaslPassword     = Environment.GetEnvironmentVariable("KAFKA_PASSWORD"),
    SslCaLocation    = "/etc/ssl/certs/internal-ca.pem",
    // If server identification fails, fix DNS — do NOT disable:
    SslEndpointIdentificationAlgorithm = SslEndpointIdentificationAlgorithm.Https,
    EnableIdempotence = true,
    Acks = Acks.All,
    CompressionType = CompressionType.Zstd
};

using var producer = new ProducerBuilder<string, string>(producerConfig)
    .SetErrorHandler((_, e) => Log.Error("Kafka error: {Reason}", e.Reason))
    .Build();
```

### mTLS

```csharp
var config = new ConsumerConfig
{
    BootstrapServers = "kafka.example.com:9093",
    SecurityProtocol = SecurityProtocol.Ssl,
    SslCaLocation         = "/etc/ssl/internal-ca.pem",
    SslCertificateLocation = "/etc/ssl/client-cert.pem",
    SslKeyLocation         = "/etc/ssl/client-key.pem",
    SslKeyPassword         = Environment.GetEnvironmentVariable("KAFKA_KEY_PW"),
    GroupId = "orders-consumer",
    EnableAutoCommit = false,
    IsolationLevel = IsolationLevel.ReadCommitted
};
```

### SASL/OAUTHBEARER (OIDC client_credentials) with a custom token refresher

librdkafka doesn't speak OIDC natively, so you hook `OAuthBearerTokenRefreshHandler`:

```csharp
var config = new ProducerConfig
{
    BootstrapServers = "kafka.example.com:9093",
    SecurityProtocol = SecurityProtocol.SaslSsl,
    SaslMechanism    = SaslMechanism.OAuthBearer,
    SaslOauthbearerConfig = "principalClaimName=sub"
};

var producer = new ProducerBuilder<string, string>(config)
    .SetOAuthBearerTokenRefreshHandler(async (client, cfg) =>
    {
        var token = await _oidc.GetClientCredentialsTokenAsync(
            clientId: "svc-orders",
            clientSecret: _secrets.KafkaClientSecret,
            scope: "kafka");

        client.OAuthBearerSetToken(
            tokenValue: token.AccessToken,
            lifetimeMs: DateTimeOffset.UtcNow.AddSeconds(token.ExpiresIn).ToUnixTimeMilliseconds(),
            principalName: token.Subject);
    })
    .Build();
```

Important: re-authentication requires `connections.max.reauth.ms` on the broker **and** the refresher firing before expiry; otherwise the broker silently kills the connection mid-produce.

## Senior-level gotchas

1. **`allow.everyone.if.no.acl.found=true` is a landmine.** Many tutorials recommend it "to get started". In production it turns authentication into a no-op for any resource without an explicit ACL.
2. **Principal name drift.** Change `ssl.principal.mapping.rules` or `sasl.oauthbearer.sub.claim.name` and all your ACLs silently stop matching. Diff before/after with `kafka-acls --list`.
3. **OIDC clock skew.** JWT `exp` + broker NTP drift → spurious `SaslAuthenticationException: invalid token`. Enforce NTP; allow 60–120s skew via `sasl.oauthbearer.clock.skew.seconds`.
4. **Re-authentication**: without `connections.max.reauth.ms`, long-lived consumer connections outlive tokens silently. Symptom: producer sends stop working after exactly token-lifetime minutes.
5. **Throttled producers look like slow brokers.** Always check `throttle-time-avg` on the client before chasing broker latency.
6. **Kafka Connect secrets.** Don't put SASL passwords in plaintext Connect configs — use `config.providers` (FileConfigProvider, Vault, AWS Secrets Manager provider).
7. **MirrorMaker 2 identity.** MM2 needs both a source-cluster principal and a target-cluster principal, each with their own ACLs (`Read` on source topics + `Write`+`Create` on target). Consolidating into one OIDC client ID complicates audit.
8. **Quota granularity.** A per-user quota covers *all* of that user's clients. If you want to cap a specific microservice replica, quota on `(user, client-id)`.
9. **Revocation.** Removing an OAuth client from the IdP does not terminate existing Kafka connections — they remain valid until token expiry *or* the next forced re-auth. Use short `connections.max.reauth.ms` (e.g. 15 min) when revocation speed matters.
10. **KRaft controller principals must be in `super.users`**. Forgetting this on a fresh cluster bricks the bootstrap — the controllers can't authorize themselves to write to `__cluster_metadata`.
11. **Don't rely on Kafka for encryption at rest compliance.** It doesn't do it. Volume encryption is the pragmatic answer; client-side encryption is the only answer for genuinely sensitive payloads.
12. **SCRAM iteration count**: sticking with the default 4096 is weak against modern offline attacks. Use 8192+ on create, and rotate SHA-256 users to SHA-512.
13. **TLS hostname verification disabled by a copy-pasted example** (`ssl.endpoint.identification.algorithm=`) turns mTLS into plain TLS. Grep configs for this line regularly.

## References

- [Kafka Authentication using SASL (Apache)](https://kafka.apache.org/41/security/authentication-using-sasl/)
- [KIP-255: OAuth Authentication via SASL/OAUTHBEARER](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=75968876)
- [KIP-768: Extend SASL/OAUTHBEARER with OIDC](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=186877575)
- [KIP-368: Allow SASL Connections to Periodically Re-Authenticate](https://cwiki.apache.org/confluence/display/KAFKA/KIP-368)
- [Confluent — Kafka Authorization ACLs overview](https://docs.confluent.io/platform/current/security/authorization/acls/overview.html)
- [Confluent — RBAC with mTLS for Kafka Brokers](https://docs.confluent.io/platform/current/kafka/configure-mds/mutual-tls-auth-rbac.html)
- [Confluent Cloud — RBAC vs ACLs](https://developer.confluent.io/courses/cloud-security/rbac-and-acls/)
- [Conduktor — Kafka ACLs and Authorization Patterns](https://www.conduktor.io/glossary/kafka-acls-and-authorization-patterns)
