# gRPC Connector - Architectural Design

## Goal

Add a gRPC connector to the Infinispan Server single-port endpoint. gRPC requests arriving on the
shared port must be detected and routed to a gRPC handler, while regular HTTP/1.1 and HTTP/2
requests continue to be handled by the REST connector.

## Implementation approach

**Strategy C (Native + ProtoStream)**: implement the gRPC wire protocol directly on top of
Infinispan's existing Netty HTTP/2 pipeline, using ProtoStream for protobuf serialization.
No grpc-java dependency. See [pipeline-integration.md](pipeline-integration.md) for details.

## Design documents

| Document | Scope |
|----------|-------|
| [pipeline-integration.md](pipeline-integration.md) | Single-port detection, Netty pipeline layout, ALPNHandler change, implementation strategies (A/B/C), recommendation, HTTP/3 readiness |
| [authentication.md](authentication.md) | Elytron integration, supported mechanisms (mTLS, Bearer, Basic), credential caching, auth flow |
| [topology.md](topology.md) | Topology and consistent-hash awareness, trailer signaling, GetTopology/SubscribeTopology RPCs, site switching |
| [transactions.md](transactions.md) | Bidirectional streaming transaction model, server-side execution via EmbeddedTransactionManager |
| [content-negotiation.md](content-negotiation.md) | Media type headers, transcoding, connection model, operation flags |
| [listeners.md](listeners.md) | Event listeners (server-streaming), near-cache bloom filter integration |
| [streaming.md](streaming.md) | Streaming put/get for large values (client-streaming PUT, server-streaming GET) |
| [cache-operations.proto](cache-operations.proto) | Protobuf definitions: CacheService (put/get/remove/conditional, query), SchemaService (type introspection) |
| [reflection.md](reflection.md) | gRPC Server Reflection: schema exposition, FileDescriptorProto serialization, tooling support |
| [performance.md](performance.md) | Wire size analysis: gRPC vs Hot Rod byte-level comparison for Put and Get |

## Decisions

Resolved questions from the design process:

- **Module**: gRPC will live in a dedicated `server/grpc` module.
- **Services**: cache, counters, multimap, locks, plus admin (cache/counter/schema management,
  metrics). Phased rollout -- cache operations first, then expand.
- **SPNEGO/Kerberos**: nice-to-have, not required for v1. Can be added later via
  `authorization: Negotiate <token>` header.
- **Connection model**: single connection per server, multiplexed RPCs. Compatible with all
  design choices (transactions, topology, auth, content negotiation).
- **TLS/ALPN**: gRPC negotiates ALPN `h2`, same as REST. No custom ALPN token needed. In
  single-port mode, `ALPNHandler` routes `h2` to the shared HTTP/2 pipeline where content-type
  detection distinguishes gRPC from REST. No compatibility concerns.
- **ProtoStream enhancement**: full service descriptor model (`ServiceDescriptor`,
  `MethodDescriptor`) — completes ProtoStream as a full protobuf schema library, useful
  beyond gRPC.
- **`SubscribeTopology`**: not required for v1. Trailer-based polling (`x-infinispan-topology-id`
  + `GetTopology` RPC) is sufficient. `SubscribeTopology` and server-initiated site switching
  can be added in a later phase.
- **Transaction isolation level**: `TxBegin` accepts an optional isolation level parameter,
  defaults to `REPEATABLE_READ`. Unsupported levels return `INVALID_ARGUMENT`. Future-proofs
  the API.
- **Transaction multi-cache scope**: per-operation `cache_name` on the `TxRequest` wrapper
  message. No dedicated `TxPut`/`TxGet` message types — reuse existing `PutRequest`,
  `GetRequest`, etc. inside a `TxRequest` `oneof` envelope.
- **`SETTINGS_MAX_CONCURRENT_STREAMS`**: default 100, matching the gRPC ecosystem standard.
