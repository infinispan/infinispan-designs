# Streaming Put / Get
## Streaming Put / Get

### Why normal unary RPCs are insufficient

Protobuf messages are fully materialised in memory before processing — the receiver
must deserialise the entire message before it can act on any part of it. For large cache
values (tens or hundreds of MB), this means:

1. **Memory pressure**: the full value exists as a byte array inside a protobuf message
   on both sender and receiver.
2. **gRPC message size limits**: the default max message size is 4 MB. Raising it globally
   undermines the protection it provides for all other RPCs.
3. **No incremental progress**: the sender must buffer the complete message before
   transmission begins — no backpressure, no progress feedback.

These are the same problems that led Hot Rod to introduce dedicated streaming opcodes
(`GET_STREAM_REQUEST` / `PUT_STREAM_REQUEST`) with explicit `START` / `NEXT` / `END`
chunking and server-side `StreamingManager` session tracking.

**Streaming put/get must be separate RPC definitions** — they cannot be handled by the
normal unary `CachePut` / `CacheGet` variants.

### How Hot Rod handles it

Hot Rod uses a multi-message protocol with explicit stream IDs:

- **PUT stream**: `START_PUT_STREAM` (sends key + metadata + first chunk) →
  `NEXT_PUT_STREAM` (subsequent chunks) → `END_PUT_STREAM` (final signal). The server
  accumulates chunks in a `CompositeByteBuf` (`PutStreamingState`) and performs the
  actual cache put on `END`.
- **GET stream**: `GET_STREAM_REQUEST` (sends key + batch size) → server responds with
  chunks of `batchAmount` bytes each, with a `complete` flag on the final chunk.
  The server wraps the full value in a `ByteBuf` (`GetStreamingState`) and slices it.
- **Session management**: `StreamingManager` tracks active streams with a 5-minute
  inactivity timeout. Each stream has an integer ID.
- **Default batch size**: 8192 bytes (client-configurable).
- **Client API**: `StreamingRemoteCache<K>` exposes `InputStream` for GET and
  `OutputStream` for PUT — standard Java I/O abstractions over the chunked protocol.

### gRPC design: streaming RPCs

gRPC's streaming primitives map directly to the Hot Rod model without the need for
explicit stream IDs or session management:

| Operation | gRPC streaming type | Rationale |
|-----------|--------------------|-----------|
| **Streaming PUT** | Client-streaming | Client sends metadata first, then value chunks, then half-closes. Server responds once with the result. |
| **Streaming GET** | Server-streaming | Client sends key + options. Server sends metadata first, then value chunks, then completes. |

#### Proto definitions

```protobuf
// ---------- Streaming put ----------

message CacheStreamingPutRequest {
  oneof payload {
    // First message: metadata (must be sent first)
    CacheStreamingPutMeta metadata = 1;
    // Subsequent messages: value chunks
    bytes chunk = 2;
  }
}

message CacheStreamingPutMeta {
  string cache_name = 1;
  bytes key = 2;
  int64 lifespan_ms = 3;
  int64 max_idle_ms = 4;
  int32 flags = 5;
  // For conditional replace — version of the entry to replace.
  int64 version = 6;
}

message CacheStreamingPutResponse {
  // Previous value only if FORCE_RETURN_VALUE flag was set.
  // For streaming puts, this is the full previous value (not chunked) —
  // if the previous value is also large, the client should use
  // a streaming get beforehand.
  optional bytes previous_value = 1;
}

// ---------- Streaming get ----------

message CacheStreamingGetRequest {
  string cache_name = 1;
  bytes key = 2;
  int32 flags = 3;
}

message CacheStreamingGetResponse {
  oneof payload {
    // First message: metadata
    CacheStreamingGetMeta metadata = 1;
    // Subsequent messages: value chunks
    bytes chunk = 2;
  }
}

message CacheStreamingGetMeta {
  int64 version = 1;
  int64 lifespan_ms = 2;
  int64 max_idle_ms = 3;
  // Total value size in bytes, if known. Allows the client to
  // pre-allocate a buffer.
  int64 total_size = 4;
}

service CacheStreamingService {
  // Client streams value chunks; server responds after the final chunk.
  rpc StreamingPut(stream CacheStreamingPutRequest) returns (CacheStreamingPutResponse);

  // Client sends key; server streams value chunks.
  rpc StreamingGet(CacheStreamingGetRequest) returns (stream CacheStreamingGetResponse);
}
```

### Chunking strategy

- **Chunk size**: client-configurable, default 8192 bytes (matching Hot Rod's default).
  Each `chunk` field in the protobuf message contains one chunk.
- **Protobuf overhead**: each chunk message has a small protobuf framing overhead
  (~5-10 bytes for the `oneof` tag + length prefix), plus the gRPC 5-byte length-prefixed
  message frame. For 8 KB chunks this is < 0.2% overhead.
- **Server-side accumulation for PUT**: the server can use a `CompositeByteBuf` (as Hot Rod
  does) to accumulate chunks without copying, then perform the cache put when the client
  half-closes the stream.
- **Server-side slicing for GET**: the server retrieves the full value, then slices it into
  chunks of the requested size and sends them as individual stream messages.

### Why not use HTTP/2 DATA frame streaming directly?

An alternative would be to bypass protobuf framing entirely and stream raw bytes over
HTTP/2 DATA frames (similar to how gRPC itself frames messages). This was rejected
because:

1. **Metadata framing**: the first message in each direction carries metadata (key,
   version, expiration). Using protobuf `oneof` gives this structure naturally.
2. **Consistency**: all other gRPC operations use protobuf messages. A raw-bytes
   exception would complicate the handler pipeline and client libraries.
3. **Observability**: protobuf messages are introspectable by gRPC middleware
   (interceptors, metrics). Raw bytes are opaque.

### Interaction with content negotiation

The `x-infinispan-value-media-type` metadata header applies to streaming operations too.
Each chunk is a fragment of the value in the negotiated media type — the server transcodes
the full value before chunking for GET, and the client sends chunks in the declared media
type for PUT. No per-chunk media type negotiation.

### Comparison with Hot Rod streaming

| Aspect | Hot Rod | gRPC |
|--------|---------|------|
| **Session management** | Explicit stream ID + `StreamingManager` with 5-min timeout | Implicit — gRPC stream lifetime is the session |
| **Chunk protocol** | `START` / `NEXT` / `END` opcodes | Client half-close (PUT), server stream completion (GET) |
| **Metadata** | Encoded in `START` message | First `oneof` variant in the stream |
| **Backpressure** | Channel writability checks | HTTP/2 flow control (automatic) |
| **Timeout** | 5-minute inactivity timeout | gRPC deadline / keepalive (configurable) |
| **Client API** | `InputStream` / `OutputStream` wrappers | gRPC `StreamObserver` — client libraries can wrap as `InputStream` / `OutputStream` |
| **Abort** | Close channel or timeout | Cancel RPC (`RST_STREAM`) — immediate, no timeout needed |

