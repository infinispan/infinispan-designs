# Wire Size Analysis: gRPC vs Hot Rod

This document compares the effective bytes-on-the-wire for Put and Get operations between
the proposed gRPC protocol and Hot Rod 4.x. All byte counts are for application-layer
payload only (excluding TCP/IP and TLS framing).

## Encoding primer

### Hot Rod encoding

Hot Rod uses a compact binary format:

- **VInt/VLong**: protobuf-style variable-length integers (7 bits per byte, MSB = continuation).
  1 byte for 0-127, 2 bytes for 128-16383, etc.
- **Strings**: VInt(length) + UTF-8 bytes.
- **Arrays** (keys/values): VInt(length) + raw bytes.
- **Fixed integers**: `long` = 8 bytes, `int` = 4 bytes, `byte` = 1 byte.

### gRPC/protobuf encoding

gRPC wraps each protobuf message in HTTP/2 frames with additional framing:

- **HTTP/2 HEADERS frame**: pseudo-headers (`:method`, `:path`, `:scheme`, `:status`) +
  custom headers, encoded via HPACK. After the initial request, HPACK's dynamic table
  compresses repeated headers dramatically (typically 1-2 bytes per repeated header via
  index reference).
- **gRPC 5-byte prefix**: 1 byte compressed flag + 4 bytes message length.
- **Protobuf message**: field tag (1 byte for field numbers 1-15) + wire type + value.
  `bytes` fields: tag + varint(length) + data. `int64`/`int32`: tag + varint.
  `optional` fields absent from the wire when unset (zero bytes).
- **HTTP/2 TRAILERS frame**: `grpc-status` + optional `grpc-message` + custom trailers.

### HTTP/2 frame overhead

Each HTTP/2 frame has a 9-byte header: length (3 bytes) + type (1) + flags (1) +
stream ID (4). A typical gRPC unary RPC uses 3 frames (HEADERS + DATA + TRAILERS on
the response side, HEADERS + DATA on the request side).

## Scenario definitions

All comparisons use these representative payloads:

| Parameter         | Value       | Notes                                   |
|-------------------|-------------|-----------------------------------------|
| Cache name        | `"myCache"` | 7 bytes UTF-8                           |
| Key               | 16 bytes    | Typical UUID or composite key           |
| Value             | 128 bytes   | Typical small entity                    |
| Topology ID       | `42`        | Fits in 1-byte VInt                     |
| Message ID        | `1`         | Fits in 1-byte VLong                    |
| Flags             | `0`         | No flags set (1-byte VInt)              |
| Version (Hot Rod) | `0x28`      | Protocol 4.0                            |
| Media types       | Default     | ProtoStream (omitted in both protocols) |
| Topology change   | No          | No topology update in response          |
| Previous value    | None        | Default IGNORE_RETURN_VALUES            |

## Put request

### Hot Rod PUT request (0x01)

```
Field                    Bytes    Encoding
-----                    -----    --------
Magic                     1       0xA0
Message ID                1       VLong(1)
Version                   1       0x28
Operation                 1       0x01 (PUT)
Cache name                8       VInt(7) + "myCache"
Flags                     1       VInt(0)
Intelligence              1       0x03 (HASH_DISTRIBUTION_AWARE)
Topology ID               1       VInt(42)
Key media type            1       0x00 (null/default, version >= 2.8)
Value media type          1       0x00 (null/default)
Other params              1       VInt(0) (empty map, version >= 4.0)
Key                      17       VInt(16) + 16 bytes
TimeUnits                 1       0x77 (both infinite)
Value                   129       VInt(128) + 128 bytes
                        ---
TOTAL                   166 bytes
```

### gRPC Put request

```
Layer                    Bytes    Notes
-----                    -----    -----
HTTP/2 HEADERS frame
  Frame header             9     3 len + 1 type + 1 flags + 4 stream ID
  :method POST             1     HPACK indexed (after first request)
  :path                   ~40    /org.infinispan.grpc.v1.CacheService/Put
                                 (first request ~40; subsequent ~1-2 via HPACK index)
  :scheme http             1     HPACK indexed
  content-type             1     HPACK indexed (after first)
  te trailers              1     HPACK indexed (after first)
  x-infinispan-cache       9     HPACK literal (first), ~2 subsequent
  x-infinispan-topology-id 3     HPACK literal (first), ~2 subsequent
  (First request total)  ~65
  (Subsequent requests)  ~15     Most headers are HPACK-indexed

HTTP/2 DATA frame
  Frame header             9
  gRPC prefix              5     1 compressed flag + 4 length
  Protobuf message:
    field 1 (key)         18     tag(1) + varint(16)(1) + 16 bytes
    field 2 (value)      131     tag(1) + varint(128)(2) + 128 bytes
    field 3 (lifespan)     0     default 0, omitted
    field 4 (max_idle)     0     default 0, omitted
    field 5 (flags)        0     default 0, omitted
  Protobuf subtotal      149
  DATA frame total       163
                         ---
TOTAL (first request)   ~228 bytes  (HEADERS ~65 + DATA 163)
TOTAL (subsequent)      ~178 bytes  (HEADERS ~15 + DATA 163)
```

### Put request comparison

| Component               | Hot Rod | gRPC (first) | gRPC (subsequent) |
|-------------------------|---------|--------------|-------------------|
| Header/framing          | 18      | ~74          | ~24               |
| Key                     | 17      | 18           | 18                |
| Value                   | 129     | 131          | 131               |
| Expiration              | 1       | 0            | 0                 |
| **Total**               | **166** | **~228**     | **~178**          |
| **Overhead vs Hot Rod** | --      | **+37%**     | **+7%**           |

After HPACK compression kicks in (typically after the first request on a connection),
the gRPC overhead converges to ~7% over Hot Rod -- about 12 extra bytes from HTTP/2
frame headers and the gRPC 5-byte prefix.

## Put response

### Hot Rod PUT response (no previous value)

```
Field                    Bytes    Encoding
-----                    -----    --------
Magic                     1       0xA1
Message ID                1       VLong(1)
Response op               1       0x02
Status                    1       0x00 (Success)
Topology flag             1       0x00 (no change)
                        ---
TOTAL                     5 bytes
```

### gRPC Put response

```
Layer                    Bytes    Notes
-----                    -----    -----
HTTP/2 HEADERS frame
  Frame header             9
  :status 200              1     HPACK indexed
  content-type             1     HPACK indexed
                         ~11

HTTP/2 DATA frame
  Frame header             9
  gRPC prefix              5
  Protobuf message:        0     PutResponse is empty (no previous value,
                                 no FORCE_RETURN_VALUE flag)
                          14

HTTP/2 TRAILERS frame
  Frame header             9
  grpc-status 0            2     HPACK literal (first), ~1 subsequent
  x-infinispan-topology-id 3     HPACK literal (first), ~2 subsequent
                         ~14     (first), ~12 (subsequent)
                         ---
TOTAL (first)            ~39 bytes
TOTAL (subsequent)       ~37 bytes
```

### Put response comparison

| Component      | Hot Rod | gRPC (first)  | gRPC (subsequent) |
|----------------|---------|---------------|-------------------|
| Header/framing | 5       | ~39           | ~37               |
| Payload        | 0       | 0             | 0                 |
| **Total**      | **5**   | **~39**       | **~37**           |
| **Overhead**   | --      | **+34 bytes** | **+32 bytes**     |

For empty responses, the HTTP/2 three-frame structure (HEADERS + DATA + TRAILERS) has a
fixed cost of ~27 bytes in frame headers alone (3 x 9 bytes). This is the main source of
overhead for responses that carry no data. Hot Rod's single-frame binary response is
extremely compact here.

## Get request

### Hot Rod GET_WITH_METADATA request (0x1B)

```
Field                    Bytes    Encoding
-----                    -----    --------
Magic                     1       0xA0
Message ID                1       VLong(1)
Version                   1       0x28
Operation                 1       0x1B (GET_WITH_METADATA)
Cache name                8       VInt(7) + "myCache"
Flags                     1       VInt(0)
Intelligence              1       0x03
Topology ID               1       VInt(42)
Key media type            1       0x00
Value media type          1       0x00
Other params              1       VInt(0)
Key                      17       VInt(16) + 16 bytes
                        ---
TOTAL                    35 bytes
```

### gRPC Get request

```
Layer                    Bytes    Notes
-----                    -----    -----
HTTP/2 HEADERS frame
  Frame header             9
  (headers, HPACK compressed) ~15  (subsequent requests, ~65 first)

HTTP/2 DATA frame
  Frame header             9
  gRPC prefix              5
  Protobuf message:
    field 1 (key)         18     tag(1) + varint(16)(1) + 16 bytes
    field 2 (flags)        0     default 0, omitted
  Protobuf subtotal       18
                         ---
TOTAL (first)            ~97 bytes  (HEADERS ~65 + DATA 41)
TOTAL (subsequent)       ~56 bytes  (HEADERS ~15 + DATA 41)
```

### Get request comparison

| Component               | Hot Rod | gRPC (first) | gRPC (subsequent) |
|-------------------------|---------|--------------|-------------------|
| Header/framing          | 18      | ~74 / ~24    | ~24               |
| Key                     | 17      | 18           | 18                |
| **Total**               | **35**  | **~97**      | **~56**           |
| **Overhead vs Hot Rod** | --      | **+177%**    | **+60%**          |

Get requests are small, so the fixed HTTP/2 framing cost is proportionally larger. On
steady-state connections the overhead is ~21 bytes (+60%).

## Get response (cache hit, with metadata)

### Hot Rod GET_WITH_METADATA response (immortal entry)

```
Field                    Bytes    Encoding
-----                    -----    --------
Magic                     1       0xA1
Message ID                1       VLong(1)
Response op               1       0x1C
Status                    1       0x00
Topology flag             1       0x00
Metadata flags            1       0x03 (both infinite)
Version                   8       long
Value                   129       VInt(128) + 128 bytes
                        ---
TOTAL                   143 bytes
```

### Hot Rod GET_WITH_METADATA response (with expiration)

```
Field                    Bytes    Encoding
-----                    -----    --------
Magic                     1       0xA1
Message ID                1       VLong(1)
Response op               1       0x1C
Status                    1       0x00
Topology flag             1       0x00
Metadata flags            1       0x00 (both specified)
Created                   8       long
Lifespan                  1       VInt(60)
LastUsed                  8       long
MaxIdle                   1       VInt(30)
Version                   8       long
Value                   129       VInt(128) + 128 bytes
                        ---
TOTAL                   161 bytes
```

### gRPC Get response (immortal entry)

```
Layer                    Bytes    Notes
-----                    -----    -----
HTTP/2 HEADERS frame     ~11     (subsequent)

HTTP/2 DATA frame
  Frame header             9
  gRPC prefix              5
  Protobuf message:
    field 1 (value)      131     tag(1) + varint(128)(2) + 128 bytes
    field 2 (metadata):
      EntryMetadata msg   12     tag(1) + varint(len)(1) + inner:
        f1 version         9       tag(1) + varint(version, ~8 bytes)
        f2 lifespan        0       default 0, omitted
        f3 max_idle        0       default 0, omitted
        f4 created         0       default 0, omitted
        f5 last_used       0       default 0, omitted
  Protobuf subtotal      143
                         ---
TOTAL (subsequent)      ~168 bytes
```

### gRPC Get response (with expiration)

```
Layer                    Bytes    Notes
-----                    -----    -----
HTTP/2 HEADERS frame     ~11     (subsequent)

HTTP/2 DATA frame
  Frame header             9
  gRPC prefix              5
  Protobuf message:
    field 1 (value)      131     tag(1) + varint(128)(2) + 128 bytes
    field 2 (metadata):
      EntryMetadata msg   42     tag(1) + varint(len)(1) + inner:
        f1 version         9       tag(1) + fixed64 or varint(~8)
        f2 lifespan_ms     3       tag(1) + varint(60000)(2)
        f3 max_idle_ms     3       tag(1) + varint(30000)(2)
        f4 created         9       tag(1) + varint(timestamp, ~8)
        f5 last_used       9       tag(1) + varint(timestamp, ~8)
  Protobuf subtotal      173

HTTP/2 TRAILERS frame    ~12     (subsequent)
                         ---
TOTAL (subsequent)      ~196 bytes
```

### Get response comparison (immortal entry)

| Component      | Hot Rod | gRPC (subsequent) |
|----------------|---------|-------------------|
| Header/framing | 5       | ~37               |
| Metadata       | 9       | 12                |
| Value          | 129     | 131               |
| **Total**      | **143** | **~168**          |
| **Overhead**   | --      | **+17%**          |

### Get response comparison (entry with expiration)

| Component      | Hot Rod | gRPC (subsequent) |
|----------------|---------|-------------------|
| Header/framing | 5       | ~37               |
| Metadata       | 27      | 42                |
| Value          | 129     | 131               |
| **Total**      | **161** | **~196**          |
| **Overhead**   | --      | **+22%**          |

Hot Rod's metadata encoding is more compact: it uses a flags byte to skip lifespan/maxidle
fields entirely when infinite, and encodes timestamps as fixed `long` (8 bytes) without
protobuf field tags. The gRPC format pays ~1 byte per field tag and uses varint encoding
for timestamps (which are large values, so varints are ~8-9 bytes -- no saving over
fixed-length).

## Round-trip totals

### Put round-trip (request + response)

| Protocol                  | Request | Response | **Round-trip** |
|---------------------------|---------|----------|----------------|
| Hot Rod                   | 166     | 5        | **171 bytes**  |
| gRPC (first)              | ~228    | ~39      | **~267 bytes** |
| gRPC (steady)             | ~178    | ~37      | **~215 bytes** |
| **Steady-state overhead** |         |          | **+26%**       |

### Get round-trip (request + response, immortal entry)

| Protocol                  | Request | Response | **Round-trip** |
|---------------------------|---------|----------|----------------|
| Hot Rod                   | 35      | 143      | **178 bytes**  |
| gRPC (first)              | ~97     | ~168     | **~265 bytes** |
| gRPC (steady)             | ~56     | ~168     | **~224 bytes** |
| **Steady-state overhead** |         |          | **+26%**       |

## How overhead scales with payload size

The fixed overhead (HTTP/2 framing, gRPC prefix, protobuf field tags) is constant
regardless of payload size. As values grow larger, the percentage overhead shrinks:

| Value size | Hot Rod Put RT | gRPC Put RT (steady) | Overhead |
|------------|----------------|----------------------|----------|
| 128 B      | 171            | ~215                 | +26%     |
| 1 KB       | 1,067          | ~1,111               | +4%      |
| 10 KB      | 10,267         | ~10,311              | +0.4%    |
| 100 KB     | 102,467        | ~102,511             | +0.04%   |

For payloads above ~1 KB, the wire overhead is negligible. The difference is only
meaningful for very small values with high request rates.

## Latency-relevant differences

Wire size is only one factor. Other performance-relevant differences:

| Aspect                      | Hot Rod                                                      | gRPC                                                   | Impact                                                           |
|-----------------------------|--------------------------------------------------------------|--------------------------------------------------------|------------------------------------------------------------------|
| **Header compression**      | None (custom binary)                                         | HPACK (shared dynamic table across requests)           | gRPC amortises header cost better over many requests             |
| **Topology delivery**       | Piggybacked (0 extra RTT)                                    | Trailer signal + separate RPC (1 extra RTT per change) | Hot Rod wins during topology changes; negligible in steady state |
| **TLS**                     | Custom ALPN token negotiation                                | Standard `h2` ALPN                                     | Identical TLS overhead                                           |
| **Parsing**                 | Custom binary decoder                                        | HTTP/2 + protobuf (two decode layers)                  | Hot Rod has one less decode layer; protobuf is well-optimised    |
| **Connection multiplexing** | Single request in flight per connection (pipelining limited) | Full HTTP/2 multiplexing                               | gRPC wins for concurrent operations on one connection            |
| **Connection count**        | One connection per server (but limited concurrency)          | One connection per server (unlimited concurrency)      | gRPC needs fewer connections for the same throughput             |

## Key takeaways

1. **Steady-state overhead is ~26% for small payloads** (128-byte values), dominated by
   HTTP/2 frame headers (27 bytes for 3 frames in a response vs Hot Rod's 5-byte header).

2. **Overhead becomes negligible (< 1%) for payloads above 1 KB.** The fixed framing cost
   is constant; only the payload size changes.

3. **HPACK compression is significant.** After the first few requests, repeated headers
   (`:method`, `:path`, `content-type`, cache name) compress to 1-2 bytes each. Hot Rod
   sends the full cache name on every request (VInt + bytes).

4. **Hot Rod's response format is extremely compact** for small/empty responses (5 bytes
   for an empty put response). This is hard to match with HTTP/2's three-frame structure.

5. **gRPC's multiplexing advantage compensates for wire overhead.** A single gRPC
   connection can sustain hundreds of concurrent RPCs, while Hot Rod's concurrency per
   connection is more limited. Fewer connections = less memory, fewer TLS handshakes,
   fewer file descriptors.

6. **For real-world workloads** (mixed value sizes, network latency dominating
   serialisation cost), the wire size difference is unlikely to be the bottleneck.
   The operational benefits of gRPC (standard tooling, language-agnostic clients,
   multiplexing) outweigh the ~26% overhead on small payloads.
