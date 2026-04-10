# Content Negotiation, Connection Model, and Operation Flags
## Content Negotiation

### How Infinispan handles media types today

Infinispan caches have a configured **storage encoding** for keys and values (e.g.
`application/x-protostream`, `application/json`, `application/x-java-object`). Clients can
read and write data in a **different** format from the storage format — the server transcodes
transparently via the `EncoderRegistry` and registered `Transcoder` instances.

**REST** uses standard HTTP headers:

| Header             | Direction | Purpose                                    |
|--------------------|-----------|--------------------------------------------|
| `Content-Type`     | Request   | Format of the value being sent             |
| `Accept`           | Request   | Desired format for the response value      |
| `key-content-type` | Request   | Format of the key (default: `text/plain`)  |
| `Content-Type`     | Response  | Actual format of the returned value        |

**Hot Rod** uses a `DataFormat` specification sent on the wire:

- Client declares `keyType` and `valueType` as `MediaType` objects.
- Wire encoding uses compact `MediaTypeIds` (e.g. 2 = `application/json`,
  12 = `application/x-protostream`, 13 = `text/plain`).
- The server creates an `AdvancedCache` wrapper with `DataConversion` configured for the
  requested types, transcoding to/from storage format transparently.

**Transcoding pipeline** (common to both protocols):

```
Client bytes ──[request media type]──→ Transcoder ──→ Cache storage format
Cache storage ──→ Transcoder ──[response media type]──→ Client bytes
```

Supported conversions include: `protostream ↔ json`, `protostream ↔ text/plain`,
`protostream ↔ octet-stream`, `json ↔ java-object`, `java-object ↔ text/plain`, and
others via registered transcoders.

### gRPC's natural content type

gRPC's wire format is protobuf. For Infinispan, the natural encoding for keys and values
is also protobuf. However, Infinispan's internal protobuf format — `application/x-protostream`
— uses a `WrappedMessage` envelope that adds type identity around each value. While this is
essential for the server's storage, query indexing, and cross-protocol interoperability, it is
an implementation detail that gRPC clients should not need to know about.

The **default media type for gRPC** is therefore **`application/x-protobuf`** — the de facto
standard for raw protobuf payloads (widely adopted by Google APIs, Spring, Envoy; not
IANA-registered). Clients send and receive **raw protobuf-encoded messages** directly. The
server handles `WrappedMessage` wrapping/unwrapping transparently at the byte level.

This means a gRPC client simply serializes its protobuf messages normally:

```java
// Java — natural protobuf usage, no WrappedMessage knowledge required
User user = User.newBuilder().setName("John").setAge(30).build();
PutRequest request = PutRequest.newBuilder()
      .setKey(ByteString.copyFromUtf8("user-123"))
      .setValue(user.toByteString())
      .build();
stub.put(request, headers);

// Get
GetResponse response = stub.get(getReq, headers);
User retrieved = User.parseFrom(response.getValue());
```

```python
# Python — equally straightforward
user = User(name="John", age=30)
request = PutRequest(key=b"user-123", value=user.SerializeToString())
stub.Put(request, metadata=headers)

# Get
response = stub.Get(get_req, metadata=headers)
user = User.FromString(response.value)
```

Transcoding is still needed when:

- The cache stores data in a non-protobuf format (e.g. `application/json`,
  `application/octet-stream`) and the client sends/expects protobuf.
- The client wants to send or receive data in a non-protobuf format (e.g. JSON for
  debugging, raw bytes for opaque blobs).
- Cross-protocol interoperability: data stored via REST as JSON needs to be readable via gRPC.

### Server-side `WrappedMessage` wrapping

Infinispan caches using protobuf storage internally encode values as `WrappedMessage` —
a ProtoStream envelope that pairs the raw protobuf bytes with a type identifier
(numeric type ID or fully-qualified type name). When a gRPC client sends
`application/x-protobuf` data, the server must wrap it before storing and unwrap it
before returning.

This is a **byte-level operation** — the server never parses or deserializes the inner
protobuf message. It simply prepends/appends the WrappedMessage field tags around the
raw bytes:

#### Wrapping (PUT path): raw protobuf → WrappedMessage

The WrappedMessage wire format for a complex message is:

```
field 17 (wrappedMessage):  tag [0x8A, 0x01] + varint(length) + raw message bytes
field 19 (wrappedTypeId):   tag [0x98, 0x01] + varint(typeId)
```

The server wraps by allocating a new byte array (~6 bytes larger than the original),
copying the raw bytes in, and writing the type ID tag:

```java
static byte[] wrapMessageBytes(byte[] rawProtobuf, int typeId) {
   int tagSize = 2;  // field 17: [0x8A, 0x01]
   int lenSize = computeVarIntSize(rawProtobuf.length);
   int typeTagSize = 2 + computeVarIntSize(typeId);  // field 19

   byte[] result = new byte[tagSize + lenSize + rawProtobuf.length + typeTagSize];
   int pos = 0;
   result[pos++] = (byte) 0x8A; result[pos++] = 0x01;
   pos = writeVarInt(result, pos, rawProtobuf.length);
   System.arraycopy(rawProtobuf, 0, result, pos, rawProtobuf.length);
   pos += rawProtobuf.length;
   result[pos++] = (byte) 0x98; result[pos++] = 0x01;
   writeVarInt(result, pos, typeId);
   return result;
}
```

For scalar keys (string, integer), the wrapping is even simpler — a string key is just:

```
field 9 (wrappedString): tag [0x4A] + varint(length) + UTF-8 bytes
```

#### Unwrapping (GET path): WrappedMessage → raw protobuf

The server scans for field 17's tag in the stored bytes and returns the payload slice.
With Netty's `ByteBuf`, this can be **zero-copy** via `readRetainedSlice()`:

```java
static ByteBuf unwrapMessageBytes(ByteBuf wrapped) {
   while (wrapped.isReadable()) {
      int tag = readVarInt(wrapped);
      int fieldNumber = tag >>> 3;
      int wireType = tag & 0x07;
      if (fieldNumber == 17 && wireType == 2) {
         int length = readVarInt(wrapped);
         return wrapped.readRetainedSlice(length);  // zero-copy
      }
      skipField(wrapped, wireType);
   }
   return null;
}
```

#### Cost

| Operation | Cost | Allocations |
|-----------|------|-------------|
| Wrap message (PUT) | O(n) memcpy + ~6 bytes header | 1 byte array |
| Wrap string key | O(n) memcpy + ~2 bytes header | 1 byte array |
| Unwrap message (GET) | O(1) tag scan + zero-copy slice | 0 (ByteBuf slice) |
| Unwrap string key | O(1) tag scan + zero-copy slice | 0 (ByteBuf slice) |

No protobuf parsing, no object instantiation, no `SerializationContext` lookup for the
inner message. The overhead is negligible compared to the cache operation itself.

#### Type identity resolution

The server needs the type ID (field 19) or type name (field 18) to write the
WrappedMessage wrapper. This is resolved from the `x-infinispan-value-type` metadata
header (see below). The name → ID resolution is a single `HashMap` lookup in the
`SerializationContext`.

### Proposed design: metadata-based format declaration

Media types are declared via **gRPC metadata headers**, keeping the protobuf message schema
clean and format-agnostic:

#### Request metadata

| Header                              | Default                        | Purpose                    |
|-------------------------------------|--------------------------------|----------------------------|
| `x-infinispan-key-media-type`      | `text/plain; charset=utf-8`   | Format of key bytes        |
| `x-infinispan-value-media-type`    | `application/x-protobuf`      | Format of value bytes      |
| `x-infinispan-value-type`          | _(none)_                       | Protobuf type name for value wrapping (e.g. `sample_bank_account.User`) |

The `x-infinispan-value-type` header is only required when the value media type is
`application/x-protobuf` and the cache stores data in `application/x-protostream`
(the common case). It provides the fully-qualified protobuf type name that the server
uses to construct the `WrappedMessage` envelope. Once set via a client interceptor, it
never needs to change for a cache storing a single entity type.

When the key media type is `text/plain` (the default), string keys are sent as raw UTF-8
bytes in the `key` field — no protobuf encoding needed. This is the most common key format
and is intentionally aligned with REST's default key content type.

#### Response trailers

| Trailer                             | Purpose                                       |
|-------------------------------------|-----------------------------------------------|
| `x-infinispan-value-media-type`    | Actual format of the returned value bytes     |

#### Server-side processing

The `GrpcRequestHandler` reads the media type metadata and creates an `AdvancedCache` wrapper
with the appropriate `DataConversion`, exactly as REST and Hot Rod do:

```java
MediaType keyType = metadata.getOrDefault("x-infinispan-key-media-type",
                                           MediaType.TEXT_PLAIN);
MediaType valueType = metadata.getOrDefault("x-infinispan-value-media-type",
                                             MediaType.APPLICATION_X_PROTOBUF);

// When valueType is x-protobuf and storage is x-protostream, the server
// applies byte-level WrappedMessage wrapping/unwrapping (see above).
// For other media type combinations, the existing EncoderRegistry
// transcoding pipeline handles the conversion.
```

The transcoding pipeline is identical to REST and Hot Rod — no new transcoders are needed
for non-protobuf formats. The `x-protobuf` ↔ `x-protostream` conversion is handled by the
lightweight byte-level wrapping described above, bypassing the full transcoder stack.

#### Examples

**1. Default (protobuf values, string keys) — minimal headers:**

```
Request metadata:
  x-infinispan-cache: myCache
  x-infinispan-value-type: sample_bank_account.User

Put { key: "user-123", value: <raw User protobuf bytes> }
// Server wraps value bytes in WrappedMessage with type ID for User
// Server wraps key string in WrappedMessage with wrappedString field
// Both are cheap byte-level operations
```

```
Get { key: "user-123" }
→ GetResponse { value: <raw User protobuf bytes> }
// Server unwraps WrappedMessage, returns raw protobuf bytes (zero-copy)
```

**2. JSON values (debugging / interop):**

```
Request metadata:
  x-infinispan-value-media-type: application/json

Put { key: "user-123", value: '{"name":"John","age":30}' }
// Server transcodes JSON → x-protostream via EncoderRegistry
```

```
Request metadata:
  x-infinispan-value-media-type: application/json

Get { key: "user-123" }
→ GetResponse { value: '{"name":"John","age":30}' }
// Server transcodes x-protostream → JSON
```

**3. Protobuf keys (composite key types):**

```
Request metadata:
  x-infinispan-key-media-type: application/x-protobuf
  x-infinispan-key-type: sample_bank_account.UserKey
  x-infinispan-value-type: sample_bank_account.User

Put { key: <raw UserKey protobuf bytes>, value: <raw User protobuf bytes> }
// Both key and value are byte-level wrapped with their respective type IDs
```

**4. Opaque binary (no-interpretation):**

```
Request metadata:
  x-infinispan-key-media-type: application/octet-stream
  x-infinispan-value-media-type: application/octet-stream

Put { key: <raw bytes>, value: <raw bytes> }
// Server stores raw bytes; cache encoding must be application/octet-stream
// or a transcoder must exist for the conversion
```

**5. ProtoStream (direct internal format) — advanced/expert use:**

```
Request metadata:
  x-infinispan-key-media-type: application/x-protostream
  x-infinispan-value-media-type: application/x-protostream

Put { key: <WrappedMessage bytes>, value: <WrappedMessage bytes> }
// No wrapping needed — client sends the internal format directly
// Useful for ProtoStream-native clients (e.g. Java with Infinispan client SDK)
```

### Two-tier encoding model

The content negotiation design supports two encoding tiers for protobuf data, allowing
clients to choose between simplicity and maximum performance:

| Tier | Media type | Default | Who wraps | Server cost | Client complexity |
|------|-----------|---------|-----------|-------------|-------------------|
| **Ergonomic** | `application/x-protobuf` | Yes | Server | O(n) memcpy per put, O(1) scan per get | None — raw protobuf |
| **Optimized** | `application/x-protostream` | No | Client | Zero | Trivial byte manipulation + type ID cache |

**Ergonomic tier** (default): clients send raw protobuf bytes. The server handles
WrappedMessage wrapping/unwrapping transparently. Type identity comes from the
`x-infinispan-value-type` metadata header. This is the right choice for ad-hoc gRPC
clients, debugging, and polyglot environments where clients should not need Infinispan-
specific knowledge.

**Optimized tier**: clients obtain type ID mappings from the server (once, at connect time),
perform the WrappedMessage wrapping themselves, and send `application/x-protostream`
data. The server stores and returns bytes with zero conversion — no wrapping, no
unwrapping, no allocation, no memcpy. This is the right choice for high-throughput client
SDKs.

The WrappedMessage wrapping is trivial byte manipulation in any language — it does not
require importing ProtoStream or understanding the full WrappedMessage proto schema:

```python
# Python — complete wrapping implementation
def wrap_message(raw_protobuf: bytes, type_id: int) -> bytes:
    """Wrap raw protobuf bytes in WrappedMessage wire format."""
    buf = bytearray()
    buf += b'\x8a\x01'                       # field 17 tag (wrappedMessage)
    buf += _encode_varint(len(raw_protobuf))  # length prefix
    buf += raw_protobuf                       # payload (copied, not parsed)
    buf += b'\x98\x01'                       # field 19 tag (wrappedTypeId)
    buf += _encode_varint(type_id)            # type ID
    return bytes(buf)

def wrap_string_key(key: str) -> bytes:
    """Wrap a string key in WrappedMessage wire format."""
    utf8 = key.encode('utf-8')
    buf = bytearray()
    buf += b'\x4a'                            # field 9 tag (wrappedString)
    buf += _encode_varint(len(utf8))
    buf += utf8
    return bytes(buf)

def unwrap_message(wrapped: bytes) -> bytes:
    """Extract raw protobuf bytes from WrappedMessage."""
    pos = 0
    while pos < len(wrapped):
        tag, pos = _decode_varint(wrapped, pos)
        field_number = tag >> 3
        wire_type = tag & 0x07
        if field_number == 17 and wire_type == 2:
            length, pos = _decode_varint(wrapped, pos)
            return wrapped[pos:pos + length]  # slice, no copy
        pos = _skip_field(wrapped, pos, wire_type)
    return None
```

A client SDK would wrap these three functions into transparent interceptors — the
application code looks identical to the ergonomic tier:

```python
# Application code — same API regardless of tier
cache = client.cache("myCache", value_type=User)
cache.put("user-123", User(name="John", age=30))
user = cache.get("user-123")
```

#### Obtaining type ID mappings

Clients in the optimized tier need a `type_name → type_id` mapping to perform wrapping.
The server exposes this via a `GetTypes` RPC in the `SchemaService`:

```protobuf
service SchemaService {
  // Returns all registered protobuf type name → type ID mappings.
  // Called once at client connect time; results are cached locally.
  // Only user-registered types are returned (not internal Infinispan types).
  rpc GetTypes(GetTypesRequest) returns (GetTypesResponse);
}

message GetTypesRequest {
  // Empty — returns all registered types.
}

message GetTypesResponse {
  // Map of fully-qualified protobuf type name → numeric type ID.
  // Example: {"sample_bank_account.User": 42, "sample_bank_account.Transaction": 43}
  map<string, int32> types = 1;
}
```

This is backed by the existing `SerializationContext.getGenericDescriptors()` API, which
already maintains the type registry. The REST equivalent is
`GET /v2/schemas?action=types`.

The mapping is stable for the lifetime of a deployment — type IDs are assigned at schema
registration time and never change (this is a ProtoStream compatibility guarantee). Clients
can cache the mapping indefinitely and only refresh if schema registration changes (which
can be signalled via a topology or schema version change).

#### Client connect flow (optimized tier)

```
1. Establish connection, authenticate
2. Call SchemaService.GetTypes()  →  {"sample_bank_account.User": 42, ...}
3. Cache type_name → type_id map locally
4. Set channel-level interceptor:
   - x-infinispan-key-media-type: application/x-protostream
   - x-infinispan-value-media-type: application/x-protostream
5. On Put: wrap_message(user.serialize(), type_ids["sample_bank_account.User"])
6. On Get: unwrap_message(response.value) → user.deserialize(raw_bytes)
```

Steps 2-4 happen once per connection. Steps 5-6 are the hot path — zero server-side
conversion.

### Alternative approaches considered

#### Option A: `application/x-protostream` as default (WrappedMessage-native)

Clients send keys and values pre-wrapped in ProtoStream's `WrappedMessage` format. This is
the internal storage format — the server stores bytes directly with zero conversion.

**Pros:**
- Zero conversion overhead — bytes flow through the server untouched
- Client has full control over type identity (wrappedTypeId/wrappedTypeName)
- Matches exactly what Hot Rod clients do

**Cons:**
- **Poor developer experience** — every operation requires double serialization (serialize
  the message, wrap in WrappedMessage, serialize the wrapper). 5-6 lines of boilerplate
  per key/value.
- **Leaks implementation details** — non-Java clients must understand ProtoStream's
  `WrappedMessage` protobuf schema, which is not a standard format.
- **Defeats gRPC's language-agnosticism** — gRPC's value proposition is that any standard
  client in any language can connect. Requiring `WrappedMessage` knowledge means every
  language needs a WrappedMessage helper library.

**Verdict:** this approach is still available to clients that set
`x-infinispan-value-media-type: application/x-protostream` explicitly. It is the right
choice for Infinispan-native client SDKs (like a future Java gRPC client) but should not
be the default for the general gRPC API.

#### Option C: Typed convenience fields in proto messages

Add `oneof key` with typed variants (string_key, int64_key, bytes_key) directly in the
proto messages:

```protobuf
message PutRequest {
  oneof key {
    bytes key_bytes = 1;
    string key_string = 8;
    int64 key_int = 9;
  }
  bytes value = 2;
  // ...
}
```

**Pros:**
- Eliminates key encoding boilerplate entirely for common types
- Self-documenting — the proto schema shows what key types are supported

**Cons:**
- **Schema bloat** — every operation message grows with key variants
- **Inconsistent** — solves the key side but not the value side (values are arbitrary
  protobuf messages and cannot be enumerated in the schema)
- **Couples schema to encoding** — content negotiation becomes split between proto fields
  (key type) and metadata headers (value type, media types)
- **Harder to extend** — adding new key types requires proto schema changes and client
  regeneration

### Channel-level format configuration

Sending media type headers on every RPC is redundant when a client consistently uses the same
formats. A client interceptor can set the headers once for all RPCs on a channel:

```
// Client-side interceptor (pseudocode, any language)
interceptor.setDefaultMetadata({
  "x-infinispan-key-media-type": "text/plain",
  "x-infinispan-value-media-type": "application/json"
});

// All subsequent RPCs automatically include these headers
// Individual RPCs can override by setting the header explicitly
```

This is equivalent to Hot Rod's `remoteCache.withDataFormat(format)` — a channel-level
binding that applies to all operations.

### Interaction with transactions

Within an `ExecuteTransaction` stream, the media type headers from the **stream's initial
metadata** apply to all operations in the transaction. This avoids repeating format
declarations for each operation in the stream. If the stream metadata omits the headers,
the defaults apply (`text/plain` for keys, `application/x-protobuf` for values).

### Why not put media types in the protobuf messages?

An alternative is to include `string key_media_type` and `string value_media_type` fields
directly in proto messages like `CachePutRequest`. This was rejected because:

1. **Schema bloat**: every cache operation message would carry two optional string fields
   that are almost never set (the default protobuf format is used in the vast majority of
   cases).
2. **Redundancy**: format rarely changes between RPCs on the same channel. Metadata headers
   avoid repeating the same values.
3. **Interceptor-friendly**: metadata headers can be set/read by generic client and server
   interceptors without knowledge of the specific message type. This enables format handling
   as cross-cutting middleware, separate from business logic.
4. **Consistency**: follows the same pattern as topology ID and authentication — operational
   metadata in headers, business data in messages.

### Comparison with REST and Hot Rod

| Aspect                     | REST                          | Hot Rod                        | gRPC (proposed)                                  |
|----------------------------|-------------------------------|--------------------------------|--------------------------------------------------|
| **Key format**             | `key-content-type` header     | `DataFormat.keyType`           | `x-infinispan-key-media-type`                   |
| **Value format (write)**   | `Content-Type` header         | `DataFormat.valueType`         | `x-infinispan-value-media-type`                 |
| **Value format (read)**    | `Accept` header               | `DataFormat.valueType`         | `x-infinispan-value-media-type`                 |
| **Default key format**     | `text/plain; charset=utf-8`   | Client's marshaller            | `text/plain; charset=utf-8`                     |
| **Default value format**   | `*/*` (match storage)         | Client's marshaller            | `application/x-protobuf`                        |
| **Type identity**          | N/A                           | Implicit in marshaller         | `x-infinispan-value-type` header                |
| **Internal format**        | Handled by transcoder         | Handled by DataConversion      | Byte-level WrappedMessage wrap/unwrap            |
| **Wire encoding**          | HTTP header strings           | Compact `MediaTypeIds` (ints)  | HTTP/2 header strings (HPACK compressed)         |
| **Transcoding engine**     | `EncoderRegistry`             | `EncoderRegistry`              | `EncoderRegistry` + byte-level wrapping          |
| **Per-channel binding**    | No (per-request)              | `withDataFormat()` per-cache   | Client interceptor                               |

## Connection Model

### Single connection per server, multiplexed RPCs

Clients should maintain **one HTTP/2 connection per cluster member** and multiplex all RPCs
on that connection as independent HTTP/2 streams. This is the natural operating mode for
HTTP/2 and gRPC — multiplexing is built into the framing layer.

On a single connection, a client can have simultaneously:

- Many **unary RPCs** in flight (cache get, put, remove, counter ops, admin ops) — each is
  a separate HTTP/2 stream. The HTTP/2 spec allows up to 2^31 concurrent streams; in
  practice, servers default to 100-250 (`SETTINGS_MAX_CONCURRENT_STREAMS`).
- One or more **bidirectional streaming RPCs** (transactions) — each transaction is a single
  HTTP/2 stream. A transaction stream does not block other streams.
- A long-lived **server-streaming RPC** (`SubscribeTopology`) running in the background.

None of these block each other. HTTP/2's stream multiplexing, flow control, and priority
mechanisms handle contention at the framing layer.

### Compatibility with design choices

| Design aspect             | Compatibility with single connection                      |
|---------------------------|-----------------------------------------------------------|
| **Unary cache RPCs**      | Fully compatible — each is an independent stream          |
| **Topology trailers**     | Every completing RPC delivers topology ID — more in-flight|
|                           | RPCs means more frequent topology feedback                |
| **Authentication caching**| One connection = one cached `Subject` = one identity.     |
|                           | Correct: all RPCs on a connection share the same auth     |
| **Transaction streams**   | A transaction occupies one stream; other RPCs continue    |
|                           | on parallel streams unblocked                             |
| **Content negotiation**   | Media type headers are per-stream (per-RPC); no conflict  |
| **Key-based routing**     | Client maintains one connection to each server; routes    |
|                           | RPCs to the connection for the correct primary owner      |

### Pipelining semantics

In gRPC/HTTP/2, "pipelining" means sending multiple requests without waiting for each
response — this is the default behaviour. Unlike HTTP/1.1 pipelining (which suffers from
head-of-line blocking), HTTP/2 streams are fully independent: responses can arrive in any
order, and a slow response on one stream does not block others.

A topology-aware client with connections to N servers and M concurrent RPCs routes each RPC
to the appropriate server's connection and fires it immediately. Responses arrive
asynchronously and are matched to their originating stream by stream ID.

### Connection lifecycle

```
Client startup:
  1. Connect to seed address
  2. Authenticate (credential headers on first RPC)
  3. Call GetTopology → discover cluster members
  4. Establish one connection to each member
  5. Route RPCs based on segment ownership

Node join:
  6. Detect topology change (via response trailer)
  7. Call GetTopology → learn about new member
  8. Establish connection to new member
  9. Update routing table

Node leave:
  10. Connection to departed member closes (or times out)
  11. Detect topology change
  12. Call GetTopology → updated segment ownership
  13. Reroute affected segments to new owners
```

### TCP-level head-of-line blocking

The one caveat with single-connection-per-server is TCP-level head-of-line blocking: if a
TCP packet is lost, all HTTP/2 streams on that connection stall until retransmission. In
a data-center or low-loss network environment this is rarely a problem. If it becomes one,
the client can open a small number of connections (2-3) per server rather than one, spreading
streams across them. This is a client-side tuning decision that does not affect the server
design.

## Operation Flags

### How Hot Rod handles flags

Hot Rod sends a flags bitmask as a `VInt` in every request header. The client-side flags
(`org.infinispan.client.hotrod.Flag`) are:

| Flag                         | Bit    | Purpose                                           |
|------------------------------|--------|---------------------------------------------------|
| `FORCE_RETURN_VALUE`         | 0x0001 | Return previous value on write ops (put/remove)   |
| `DEFAULT_LIFESPAN`           | 0x0002 | Use server-configured default lifespan            |
| `DEFAULT_MAXIDLE`            | 0x0004 | Use server-configured default maxIdle             |
| `SKIP_CACHE_LOAD`            | 0x0008 | Skip loading from cache stores (persistence)      |
| `SKIP_INDEXING`              | 0x0010 | Don't update query indexes for this operation     |
| `SKIP_LISTENER_NOTIFICATION` | 0x0020 | Don't fire client/server listener events          |

The server maps these to embedded cache `Flag` values:

- **`FORCE_RETURN_VALUE` absent** → server adds `Flag.IGNORE_RETURN_VALUES` on non-conditional
  write operations (put, remove). This is a critical performance optimization — it avoids
  reading the previous value from the data container.
- **`FORCE_RETURN_VALUE` present** → server omits `IGNORE_RETURN_VALUES`, and the cache
  returns the previous value. Only allowed on transactional caches (a warning is logged
  otherwise, since non-transactional caches may return stale values).
- **`SKIP_CACHE_LOAD`** → `Flag.SKIP_CACHE_LOAD`: skips persistence store reads.
- **`SKIP_INDEXING`** → `Flag.SKIP_INDEXING`: suppresses query index updates.
- **`SKIP_LISTENER_NOTIFICATION`** → `Flag.SKIP_LISTENER_NOTIFICATION`: suppresses events.

The server also validates flag applicability per operation — e.g. `SKIP_INDEXING` is only
honoured on operations marked `CAN_SKIP_INDEXING` (writes), `SKIP_CACHE_LOAD` only on
operations marked `CAN_SKIP_CACHE_LOAD`.

### Proposed gRPC design: flags in protobuf messages

For gRPC, flags are specified as **an `int32` bitmask field in the request message** rather
than in metadata headers. This is because flags are semantically tied to the operation — they
change what the operation does — not operational metadata like topology or auth.

```protobuf
message CachePutRequest {
  string cache_name = 1;
  bytes key = 2;
  bytes value = 3;
  int64 lifespan_ms = 4;     // 0 = use server default
  int64 max_idle_ms = 5;     // 0 = use server default
  int32 flags = 6;           // bitmask of operation flags
}

message CacheRemoveRequest {
  string cache_name = 1;
  bytes key = 2;
  int32 flags = 3;
}

message CacheGetRequest {
  string cache_name = 1;
  bytes key = 2;
  int32 flags = 3;           // e.g. SKIP_CACHE_LOAD
}
```

### Flag values

```protobuf
// Flag constants for the `flags` bitmask field.
// Values match org.infinispan.client.hotrod.Flag for consistency.

// By default, write operations (put, remove) do NOT return the previous
// value — the server applies IGNORE_RETURN_VALUES for performance.
// Set this flag to receive the previous value in the response.
// FORCE_RETURN_VALUE = 0x0001;

// Use the server-configured default lifespan instead of the lifespan_ms
// value in the request. When this flag is set, lifespan_ms is ignored.
// DEFAULT_LIFESPAN = 0x0002;

// Use the server-configured default maxIdle instead of the max_idle_ms
// value in the request. When this flag is set, max_idle_ms is ignored.
// DEFAULT_MAXIDLE = 0x0004;

// Skip loading from persistent cache stores. The operation only sees
// data in memory.
// SKIP_CACHE_LOAD = 0x0008;

// Do not update query indexes as a result of this operation.
// SKIP_INDEXING = 0x0010;

// Do not fire client or server listener notifications for this operation.
// SKIP_LISTENER_NOTIFICATION = 0x0020;
```

### Default behaviour: IGNORE_RETURN_VALUES

The most important flag interaction is the **default suppression of return values** on
writes. When `flags` is `0` (the default) and the operation is a non-conditional write
(put, putAll, remove without version), the server automatically applies
`Flag.IGNORE_RETURN_VALUES`. This means:

- `CachePutRequest` with `flags = 0` → `CachePutResponse` has empty `previous_value`
- `CachePutRequest` with `flags = 0x0001` → `CachePutResponse` contains the previous value

This matches Hot Rod's behaviour exactly and is a significant performance optimisation for
the common case where callers don't need the old value.

### Server-side flag validation

The server validates flags against the operation type, matching Hot Rod's `OpReqs` model:

| Flag                         | Allowed on                                       |
|------------------------------|--------------------------------------------------|
| `FORCE_RETURN_VALUE`         | put, putIfAbsent, replace, remove                |
| `SKIP_CACHE_LOAD`            | get, containsKey, put, remove, size, getAll, etc.|
| `SKIP_INDEXING`              | put, putIfAbsent, replace, remove, putAll        |
| `SKIP_LISTENER_NOTIFICATION` | All operations                                   |
| `DEFAULT_LIFESPAN`           | put, putIfAbsent, replace                        |
| `DEFAULT_MAXIDLE`            | put, putIfAbsent, replace                        |

Invalid flags for an operation are silently ignored (matching Hot Rod's behaviour).

### Interaction with transactions

Within an `ExecuteTransaction` stream, the `TxPut`, `TxGet`, `TxRemove` messages include
a `flags` field with the same semantics. The `FORCE_RETURN_VALUE` flag is particularly
useful here — within a transaction, returning previous values is always safe (the
transaction provides consistent reads).

### Why bitmask in messages, not metadata headers?

1. **Semantic coupling**: flags modify what the operation _does_ (return a value, skip a
   store). They're part of the operation's intent, not transport-level metadata.
2. **Per-operation**: different operations in the same RPC batch or transaction may need
   different flags. Message-level fields handle this naturally.
3. **Type safety**: `int32` bitmask is explicit in the proto schema. Metadata headers would
   be stringly-typed and require parsing.
4. **Consistency with Hot Rod**: same bit values, same default behaviour. Client libraries
   can share flag constant definitions.
5. **Self-documenting**: the proto schema shows which messages accept flags, and the flag
   constants are documented alongside the message definitions.

