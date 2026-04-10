# Authentication
## Authentication

### gRPC authentication patterns

gRPC defines authentication at two layers:

1. **Channel-level (transport)**: TLS and mutual TLS (mTLS). Established once at connection
   setup. All RPCs on the connection inherit the channel's security context.

2. **Per-RPC credentials (call credentials)**: Carried as HTTP/2 headers (gRPC metadata) on each
   individual RPC stream. The standard mechanism is the `authorization` header — typically
   `Bearer <token>` for JWT/OAuth2 tokens.

In the standard grpc-java library, `CallCredentials` attach metadata to every outgoing RPC.
Since we are implementing Strategy C (native), this translates to reading standard HTTP/2
headers on each request stream.

### Infinispan's existing authentication model

Each protocol has its own authenticator abstraction backed by Elytron:

| Protocol  | Authenticator type            | Elytron factory             | Auth timing           |
|-----------|-------------------------------|-----------------------------|-----------------------|
| REST/HTTP | `RestAuthenticator`           | `HttpAuthenticationFactory` | Per-request           |
| Hot Rod   | `SaslAuthenticator`           | `SaslAuthenticationFactory` | Once at connection    |
| RESP      | `RespAuthenticator`           | Direct `SecurityDomain`     | Once via AUTH command |
| Memcached | `SaslAuthenticator` or direct | `SaslAuthenticationFactory` | Once at connection    |

The server runtime (`EndpointConfigurationBuilder`) auto-configures mechanisms based on
realm features:

| Realm feature         | REST mechanism   | Hot Rod mechanism         |
|-----------------------|------------------|---------------------------|
| `TOKEN`               | `BEARER_TOKEN`   | `OAUTHBEARER`             |
| `TRUST`               | `CLIENT_CERT`    | `EXTERNAL`                |
| `PASSWORD_HASHED`     | `DIGEST-SHA-256` | `SCRAM-SHA-*`, `DIGEST-*` |
| `PASSWORD_HASHED`+TLS | `BASIC`          | `PLAIN`                   |
| Kerberos identity     | `SPNEGO`         | `GSSAPI`, `GS2-KRB5`      |

Elytron integrations live in the `server/runtime` module:

- `ElytronHTTPAuthenticator` — wraps `HttpAuthenticationFactory`, supports BASIC, BEARER,
  DIGEST, CLIENT_CERT, SPNEGO.
- `ElytronSASLAuthenticator` — wraps `SaslAuthenticationFactory`, supports PLAIN, SCRAM-*,
  DIGEST-*, EXTERNAL, OAUTHBEARER, GSSAPI.
- `ElytronUsernamePasswordAuthenticator` — direct `SecurityDomain.authenticate()` for
  simple username/password (used by RESP and text Memcached).

All paths ultimately produce a `javax.security.auth.Subject` stored in `ConnectionMetadata`,
which is then used for authorization on subsequent operations.

### Connection-level credential caching

`RestRequestHandler` already caches the authenticated `Subject` per-connection (fields
`subject` and `authorization` at lines 59-60). If the `Authorization` header on a subsequent
request matches the cached value, authentication is skipped. If it changes, the cached subject
is invalidated and re-authentication occurs.

Hot Rod and RESP go further: authentication happens once (SASL exchange or AUTH command) and
the resulting `Subject` applies to all subsequent commands on the connection.

### Proposed gRPC authentication design

gRPC authentication should follow a **hybrid model**: credentials are presented as HTTP/2
headers (metadata) on the first RPC, the authenticated `Subject` is cached at the connection
level, and subsequent RPCs on the same connection reuse it — re-authenticating only if the
credentials change.

This matches the existing `RestRequestHandler` caching pattern and is compatible with how
gRPC clients typically work (they attach the same `CallCredentials` to every RPC on a channel).

#### Supported mechanisms

| Method                 | gRPC metadata                   | Elytron mechanism | Notes                                             |
|------------------------|---------------------------------|-------------------|---------------------------------------------------|
| **Client certificate** | None (TLS layer)                | `CLIENT_CERT`     | Peer principal extracted from SSL session         |
| **Bearer token (JWT)** | `authorization: Bearer <token>` | `BEARER_TOKEN`    | Validated via `TokenSecurityRealm` (JWT/OAuth2)   |
| **Username/password**  | `authorization: Basic <base64>` | `BASIC`           | Only over TLS; uses `SecurityDomain.authenticate` |

#### How it maps to existing Elytron infrastructure

Since gRPC runs over HTTP/2, the natural fit is to **reuse `ElytronHTTPAuthenticator`** — gRPC
metadata are HTTP/2 headers, so the `authorization` header works identically to REST.

The `EndpointConfigurationBuilder` would configure gRPC mechanisms from realm features using
the same logic as REST:

```
TOKEN realm        -> BEARER_TOKEN mechanism  (authorization: Bearer <jwt>)
TRUST realm        -> CLIENT_CERT mechanism   (mTLS, no header needed)
PASSWORD_HASHED+TLS -> BASIC mechanism        (authorization: Basic <base64>)
```

This means **no new Elytron authenticator implementation is needed**. The gRPC handler delegates
to the same `ElytronHTTPAuthenticator` that REST uses, just invoked from a different point in
the pipeline.

#### Authentication flow

```
Client                          Server (GrpcRequestHandler)
  |                                  |
  |-- TLS handshake (opt. mTLS) ---->|  (1) channel-level: extract peer cert if present
  |                                  |      store in ConnectionMetadata
  |                                  |
  |-- HEADERS (:path, auth, ...) --->|  (2) first RPC: read `authorization` header
  |-- DATA (protobuf message) ----->|      authenticate via ElytronHTTPAuthenticator
  |                                  |      cache Subject + authorization in connection state
  |                                  |
  |<-- HEADERS (:status 200) --------|  (3) send response
  |<-- DATA (protobuf response) ----|
  |<-- TRAILERS (grpc-status 0) ----|
  |                                  |
  |-- HEADERS (same auth) --------->|  (4) subsequent RPC: authorization header unchanged
  |-- DATA ----------------------->|      skip re-auth, reuse cached Subject
  |                                  |
  |-- HEADERS (different auth) ---->|  (5) credential change: invalidate cache
  |-- DATA ----------------------->|      re-authenticate, cache new Subject
```

#### Authentication errors

gRPC uses standard status codes for auth failures:

| Scenario                      | gRPC status             | Trailer                                 |
|-------------------------------|-------------------------|-----------------------------------------|
| No credentials, auth required | `UNAUTHENTICATED (16)`  | `grpc-message: authentication required` |
| Invalid credentials           | `UNAUTHENTICATED (16)`  | `grpc-message: invalid credentials`     |
| Valid creds, no permission    | `PERMISSION_DENIED (7)` | `grpc-message: access denied`           |

For bearer tokens, the server should also return `www-authenticate: Bearer` in the response
trailers to signal the expected auth scheme, matching HTTP `401` semantics.

#### Per-stream sub-channel pipeline with auth

The `GrpcRequestHandler` handles authentication itself (before dispatching to the service
method), rather than relying on the REST pipeline's `AccessControlFilter` or `RestRequestHandler`
auth logic. This keeps gRPC auth self-contained while reusing the same Elytron backend:

```
per-stream sub-channel pipeline (after gRPC detection):

    [Http2StreamFrameToHttpObjectCodec]
    [AccessControlFilter]              <-- IP filtering
    [HttpObjectAggregator]             <-- assembles full request body
    [StreamCorrelatorHandler]          <-- HTTP/2 stream ID propagation
    [GrpcRequestHandler]               <-- auth + dispatch (delegates to ElytronHTTPAuthenticator)
```

REST-specific handlers (content compression, CORS, keep-alive, chunked writes) are removed
from the pipeline on gRPC detection. See "Detection point" section for details.

#### Client certificate authentication detail

For mTLS, the certificate is validated at the TLS layer by Netty's `SslHandler`. The
`GrpcRequestHandler` extracts the peer principal:

```java
SslHandler sslHandler = ctx.channel().parent().pipeline().get(SslHandler.class);
if (sslHandler != null) {
    Principal peerPrincipal = sslHandler.engine().getSession().getPeerPrincipal();
    // authenticate via CLIENT_CERT mechanism
}
```

This is identical to `ClientCertAuthenticator` in the REST module. If `requireClientAuth` is
enabled in the server's encryption configuration, the TLS handshake itself rejects clients
without valid certificates.

#### Token refresh and expiry

For JWT tokens, Elytron's `TokenSecurityRealm` validates the token (signature, expiry, issuer,
audience) on each authentication. The connection-level credential cache compares the raw
`authorization` header string — if the client refreshes its token (sends a new JWT), the cached
Subject is invalidated and re-authentication occurs with the new token. This handles token
rotation naturally.

### Authentication configuration

gRPC auth configuration should be implicit (auto-derived from realm features), matching the
existing pattern. The server configuration would look like:

```xml
<endpoints>
  <endpoint socket-binding="default" security-realm="default">
    <hotrod-connector />
    <rest-connector />
    <resp-connector />
    <grpc-connector />   <!-- NEW: mechanisms auto-configured from realm -->
  </endpoint>
</endpoints>
```

The `EndpointConfigurationBuilder` would add a gRPC case alongside the existing REST/Hot Rod
cases, mapping realm features to HTTP mechanisms:

```java
// In enableImplicitAuthentication for gRPC:
if (features.contains(TOKEN))        -> add BEARER_TOKEN
if (features.contains(TRUST))        -> add CLIENT_CERT
if (features.contains(ENCRYPT) &&
    features.contains(PASSWORD_HASHED)) -> add BASIC
```
