# Topology, Consistent-Hash Awareness, and Site Switching
## Topology and Consistent-Hash Awareness

### How Hot Rod does it

Hot Rod achieves intelligent client routing by **piggybacking topology updates on every
response**. The mechanism works as follows:

1. **Client declares its intelligence level** in each request header:
   - `BASIC (0x01)`: no topology awareness, server never sends topology.
   - `TOPOLOGY_AWARE (0x02)`: client receives the cluster member list (for failover).
   - `HASH_DISTRIBUTION_AWARE (0x03)`: client receives member list + segment ownership
     (for key-based routing to primary owners).

2. **Client sends its current topology ID** (a `VInt`) in every request header.

3. **Server compares** the client's topology ID with `cacheTopology.getReadConsistentHash()
   .hashCode()`. If they differ, the server includes a topology update in the response.

4. **Topology update payload** (for `HASH_DISTRIBUTION_AWARE`):
   ```
   topology_id:       VInt          (hashCode of the consistent hash)
   members_count:     VInt
   members[]:         string host + ushort port  (indexed 0..N-1)
   hash_function:     1 byte        (version 3 = MurmurHash3)
   num_segments:      VInt          (typically 256)
   segments[]:        for each segment:
                        owner_count: 1 byte
                        owner_indices[]: VInt[]  (references into members[])
   ```

5. **Client-side routing**: on a key-based operation the client hashes the key
   (`MurmurHash3 → normalize → divide by segment_size → segment_id`), looks up
   `segmentOwners[segment_id][0]` (primary owner), and sends the request directly
   to that server.

This design is efficient (zero extra round-trips) but tightly coupled to Hot Rod's binary
response envelope — every response includes a topology-change marker byte.

### Challenges for gRPC

gRPC responses are structured as `HEADERS + DATA + TRAILERS`. The DATA frame carries a single
protobuf message — the business response. There is no binary response header envelope like Hot
Rod's. We have three options for conveying topology:

1. **Embed in every response message** (envelope pattern): add an optional `TopologyUpdate`
   field to every response proto. Zero extra round-trips but pollutes every response schema.

2. **Use HTTP/2 trailers**: put topology metadata in gRPC trailing metadata. Clean separation
   from business data. Topology ID fits easily; full segment ownership does not (too large
   for headers).

3. **Dedicated topology RPCs**: topology is fetched explicitly. Regular RPCs only carry a
   lightweight change signal. Extra round-trip on topology change, but response messages are
   pure business data.

### Proposed design: trailer signaling + dedicated topology RPCs

The design separates **change detection** (lightweight, every RPC) from **topology transfer**
(heavyweight, on-demand). This is idiomatic gRPC and works well with client interceptors
across all languages.

#### Request metadata (sent by client on every RPC)

| Header                            | Value     | Purpose                              |
|-----------------------------------|-----------|--------------------------------------|
| `x-infinispan-topology-id`       | integer   | Client's current topology ID         |
| `x-infinispan-cache`             | string    | Cache name (topology is per-cache)   |

If the client is not topology-aware, these headers are omitted and the server never includes
topology trailers.

#### Response trailers (sent by server on every RPC)

| Trailer                           | Value     | Condition                            |
|-----------------------------------|-----------|--------------------------------------|
| `x-infinispan-topology-id`       | integer   | Always (current server topology ID)  |

The server computes the current topology ID as `cacheTopology.getReadConsistentHash()
.hashCode()` — identical to Hot Rod. Including it in every response trailer is cheap (one
small header). The client compares it with its cached value after every RPC.

#### Topology RPCs

```protobuf
service Topology {
  // Fetch full topology for a cache (members + segment ownership).
  // Called by the client when it detects a topology ID change via
  // the x-infinispan-topology-id response trailer.
  rpc GetTopology(TopologyRequest) returns (TopologyResponse);

  // Subscribe to topology changes for a cache. The server sends a
  // TopologyResponse each time the topology changes. Optional —
  // clients can use polling via GetTopology instead.
  rpc SubscribeTopology(TopologyRequest) returns (stream TopologyResponse);
}

message TopologyRequest {
  // Cache name. Required — topology is per-cache.
  string cache_name = 1;
}

message ServerAddress {
  string host = 1;
  int32 port = 2;
}

message SegmentOwnership {
  // Indices into the members list of the enclosing TopologyResponse.
  repeated int32 owner_indices = 1;
}

message TopologyResponse {
  // Topology ID (hashCode of the consistent hash).
  int32 topology_id = 1;

  // Cluster members that own segments in this cache.
  repeated ServerAddress members = 2;

  // Hash function version (3 = MurmurHash3).
  int32 hash_function_version = 3;

  // Segment ownership. Length = num_segments.
  // segments[i].owner_indices are indices into the members list.
  // First owner (index 0) is the primary owner.
  repeated SegmentOwnership segments = 4;
}
```

#### Full flow

```
Client                                 Server
  |                                       |
  |  (1) Initial: no topology yet         |
  |  GetTopology(cache="myCache") ------->|
  |<--- TopologyResponse {               |
  |       topology_id: 42,               |
  |       members: [A:11222, B:11222],   |
  |       segments: [{0},{1},{0,1},...]}  |
  |                                       |
  |  Client builds segment ownership map  |
  |  Hash key → segment → primary owner  |
  |                                       |
  |  (2) Normal RPC, routed to owner      |
  |  CacheGet(key="user:1")              |
  |  headers:                             |
  |    x-infinispan-topology-id: 42      |
  |    x-infinispan-cache: myCache       |
  |  ----(routed to primary owner A)---->|
  |<--- DATA: CacheGetResponse           |
  |     trailers:                         |
  |       x-infinispan-topology-id: 42   |
  |                                       |
  |  Client: 42 == 42, no change          |
  |                                       |
  |  (3) Topology changes (node C joins)  |
  |  CacheGet(key="user:2")              |
  |  headers:                             |
  |    x-infinispan-topology-id: 42      |
  |  ----(routed to B, still valid)----->|
  |<--- DATA: CacheGetResponse           |
  |     trailers:                         |
  |       x-infinispan-topology-id: 57   |  ← changed!
  |                                       |
  |  Client: 42 != 57, refresh topology   |
  |  GetTopology(cache="myCache") ------->|
  |<--- TopologyResponse {               |
  |       topology_id: 57,               |
  |       members: [A, B, C],            |
  |       segments: [{0},{1},{2},...]}    |
  |                                       |
  |  Client updates routing table         |
  |  Subsequent RPCs use new topology     |
```

#### Comparison with Hot Rod's approach

| Aspect                      | Hot Rod                       | gRPC (proposed)                    |
|-----------------------------|-------------------------------|------------------------------------|
| **Change detection**        | Topology marker in response   | `x-infinispan-topology-id` trailer |
| **Topology transfer**       | Piggybacked on response       | Separate `GetTopology` RPC         |
| **Extra round-trip**        | None                          | 1 per topology change              |
| **Response message purity** | Embedded in binary envelope   | Clean — pure business data         |
| **Client interceptor**      | Custom binary parsing         | Standard gRPC metadata interceptor |
| **Proactive push**          | No (only on client request)   | Optional `SubscribeTopology` stream|

The extra round-trip on topology change is a minor cost — topology changes are infrequent
(node join/leave, rebalance) while RPCs are high-frequency. The operational overhead of one
extra RPC per topology change is negligible. The `SubscribeTopology` stream eliminates even
this cost for clients that want proactive updates.

#### Client-side routing

The client maintains a per-cache routing table identical to Hot Rod's `CacheInfo`:

```
CacheTopologyInfo:
  topology_id:     int
  members:         ServerAddress[]
  segment_owners:  ServerAddress[][]    (segment_id → owner list)
  num_segments:    int
  segment_size:    int                  (Integer.MAX_VALUE / num_segments)
```

**Key routing algorithm** (identical to Hot Rod's `SegmentConsistentHash`):

```
segment_id = murmurHash3(key_bytes) / segment_size
primary_owner = segment_owners[segment_id][0]
→ send RPC to primary_owner
```

The hash function (MurmurHash3, version 3) is shared between server and all clients. The
client only needs the segment ownership table to route correctly — the same table format Hot
Rod uses.

#### Handling topology staleness

Between detecting a topology change (via trailer) and fetching the new topology (via
`GetTopology`), the client may route RPCs to the wrong server. This is safe:

- The server that receives a misrouted request still processes it correctly — it just forwards
  internally to the correct owner (like Hot Rod does when a client has a stale topology).
- The performance cost is one extra internal hop for a brief window.
- Once `GetTopology` returns, the client's routing table is updated and subsequent RPCs are
  routed correctly.

#### Non-key operations

Not all RPCs are key-based. For operations like `CacheClear`, `CacheSize`, `AdminCreateCache`,
or `ClusterStatus`, key-based routing does not apply. The client can send these to any cluster
member using round-robin or a similar balancing strategy. The `x-infinispan-cache` header
still triggers topology tracking, so the client stays aware of membership changes for
failover purposes.

#### Bootstrap and initial topology

When a client connects for the first time:

1. It has no topology. It connects to one of its configured seed addresses.
2. It calls `GetTopology(cache_name)` to receive the full topology.
3. It establishes connections to all cluster members (or a subset).
4. It begins routing key-based RPCs to primary owners.

If the initial `GetTopology` call fails (e.g. seed node is down), the client tries the next
seed address — standard gRPC retry/failover behaviour.

### Building a topology/hash-aware client

This section describes how a gRPC client uses topology information to route key-based
operations directly to the primary owner, avoiding internal forwarding hops. The algorithm
is identical to what the Hot Rod Java client uses — any language can implement it.

#### The hash algorithm: MurmurHash3

Infinispan uses a custom variant of MurmurHash3 (x64, returning 32 bits) with a hardcoded
seed of `9001`. The algorithm is based on Austin Appleby's original MurmurHash3_x64_128 but
Infinispan's version uses non-standard initial state constants and a different `bmix`
function, so **standard MurmurHash3 library implementations will not produce correct
results**. Clients must port Infinispan's exact implementation.

Source reference: `commons/all/src/main/java/org/infinispan/commons/hash/MurmurHash3.java`

Key characteristics:
- Processes input in 16-byte blocks, reading two little-endian `int64` values per block
- Uses custom initial state: `h1 = 0x9368e53c2f6af274 ^ seed`, `h2 = 0x586dcd208f7cd3fd ^ seed`
- Uses evolving constants: `c1` and `c2` are updated each round (multiplied by 5 and added
  to fixed constants), unlike standard MurmurHash3 where they are fixed
- The `bmix` step uses rotate-left by 23 (not 31/33 as in standard MurmurHash3)
- Returns the upper 32 bits of the 64-bit `h1` result: `(h1 >>> 32)` cast to a 32-bit int

#### Key-to-segment routing

Given a key (as raw bytes), the segment is computed in three steps:

```
1. hash           = murmurHash3_x64_32(key_bytes, seed=9001)
2. normalized     = hash & 0x7FFFFFFF          # mask sign bit → positive int
3. segment_id     = normalized // segment_size
```

Where `segment_size` is derived from the number of segments (provided in `TopologyResponse`):

```
segment_size = ceil(2^31 / num_segments)
```

Once the segment is known, the primary owner is looked up from the topology:

```
primary_owner = topology.segments[segment_id].owner_indices[0]
server        = topology.members[primary_owner]
```

#### Key encoding

In the gRPC protocol, keys are opaque `bytes` fields in request messages (e.g.,
`PutRequest.key`, `GetRequest.key`). The default key media type is `text/plain`, meaning
keys are UTF-8 encoded strings. The client hashes these exact bytes — the same bytes sent
in the request — to determine the segment. No additional wrapping or encoding is applied
before hashing.

If a different key media type is used (via `x-infinispan-key-media-type`), the client must
hash the bytes in the wire format that matches the server's storage encoding for correct
routing. When key media types differ between client and server, the server transcodes
internally, but routing based on the client-side bytes may not match the server's segment
assignment. For consistent routing, use the cache's storage encoding for keys.

#### Connection management

A topology/hash-aware client maintains one gRPC channel (HTTP/2 connection) per cluster
member. On receiving a `TopologyResponse`, the client:

1. Opens channels to any new members not yet connected.
2. Closes channels to members that have left the cluster.
3. Updates the local segment ownership table.

For key-based RPCs, the client selects the channel corresponding to the primary owner.
For non-key RPCs (e.g., `Size`, `Clear`, `Query`), any channel can be used with
round-robin or similar balancing.

#### Example: topology-aware client in Python

The following example demonstrates a minimal topology/hash-aware gRPC client. It
bootstraps by fetching the topology, implements Infinispan's MurmurHash3 for key routing,
and detects topology changes via response trailers.

```python
import struct
import math
import grpc

# -- Generated from cache-operations.proto and topology section --
# from infinispan_grpc_v1 import cache_pb2, cache_pb2_grpc, topology_pb2, topology_pb2_grpc
# For this example, we assume the generated stubs are available.


# ============================================================================
# MurmurHash3 — Infinispan's custom variant (x64, 32-bit output, seed 9001)
# ============================================================================
# This is NOT standard MurmurHash3. Infinispan uses non-standard initial
# state constants and a different bmix function. You must use this exact
# implementation for correct segment routing.

_MASK64 = 0xFFFFFFFFFFFFFFFF

def _rotl64(x, r):
    return ((x << r) | (x >> (64 - r))) & _MASK64

def _fmix64(k):
    k ^= k >> 33
    k = (k * 0xff51afd7ed558ccd) & _MASK64
    k ^= k >> 33
    k = (k * 0xc4ceb9fe1a85ec53) & _MASK64
    k ^= k >> 33
    return k

def _to_signed64(v):
    """Convert unsigned 64-bit to signed for Python arithmetic consistency."""
    return v - (1 << 64) if v >= (1 << 63) else v

def murmur3_x64_32(key: bytes, seed: int = 9001) -> int:
    """Infinispan's MurmurHash3 x64 variant, returning 32-bit hash."""
    h1 = (0x9368e53c2f6af274 ^ seed) & _MASK64
    h2 = (0x586dcd208f7cd3fd ^ seed) & _MASK64
    c1 = 0x87c37b91114253d5
    c2 = 0x4cf5ad432745937f

    # Process 16-byte blocks
    nblocks = len(key) // 16
    for i in range(nblocks):
        k1 = struct.unpack_from('<q', key, i * 16)[0] & _MASK64
        k2 = struct.unpack_from('<q', key, i * 16 + 8)[0] & _MASK64

        # bmix
        k1 = (k1 * c1) & _MASK64
        k1 = _rotl64(k1, 23)
        k1 = (k1 * c2) & _MASK64
        h1 ^= k1
        h1 = (h1 + h2) & _MASK64

        h2 = _rotl64(h2, 41)

        k2 = (k2 * c2) & _MASK64
        k2 = _rotl64(k2, 23)
        k2 = (k2 * c1) & _MASK64
        h2 ^= k2
        h2 = (h2 + h1) & _MASK64

        h1 = (h1 * 3 + 0x52dce729) & _MASK64
        h2 = (h2 * 3 + 0x38495ab5) & _MASK64

        c1 = (c1 * 5 + 0x7b7d159c) & _MASK64
        c2 = (c2 * 5 + 0x6bce6396) & _MASK64

    # Tail (remaining bytes)
    tail = nblocks * 16
    k1 = 0
    k2 = 0
    remaining = len(key) & 15

    # Fall-through switch for k2 bytes (15..9)
    if remaining >= 15: k2 ^= key[tail + 14] << 48
    if remaining >= 14: k2 ^= key[tail + 13] << 40
    if remaining >= 13: k2 ^= key[tail + 12] << 32
    if remaining >= 12: k2 ^= key[tail + 11] << 24
    if remaining >= 11: k2 ^= key[tail + 10] << 16
    if remaining >= 10: k2 ^= key[tail + 9] << 8
    if remaining >= 9:  k2 ^= key[tail + 8]

    # Fall-through switch for k1 bytes (8..1)
    if remaining >= 8: k1 ^= key[tail + 7] << 56
    if remaining >= 7: k1 ^= key[tail + 6] << 48
    if remaining >= 6: k1 ^= key[tail + 5] << 40
    if remaining >= 5: k1 ^= key[tail + 4] << 32
    if remaining >= 4: k1 ^= key[tail + 3] << 24
    if remaining >= 3: k1 ^= key[tail + 2] << 16
    if remaining >= 2: k1 ^= key[tail + 1] << 8
    if remaining >= 1: k1 ^= key[tail + 0]

    if remaining > 0:
        # bmix for tail
        k1 = (k1 * c1) & _MASK64
        k1 = _rotl64(k1, 23)
        k1 = (k1 * c2) & _MASK64
        h1 ^= k1
        h1 = (h1 + h2) & _MASK64

        h2 = _rotl64(h2, 41)

        k2 = (k2 * c2) & _MASK64
        k2 = _rotl64(k2, 23)
        k2 = (k2 * c1) & _MASK64
        h2 ^= k2
        h2 = (h2 + h1) & _MASK64

        h1 = (h1 * 3 + 0x52dce729) & _MASK64
        h2 = (h2 * 3 + 0x38495ab5) & _MASK64

        c1 = (c1 * 5 + 0x7b7d159c) & _MASK64
        c2 = (c2 * 5 + 0x6bce6396) & _MASK64

    # Finalization
    h2 ^= len(key)

    h1 = (h1 + h2) & _MASK64
    h2 = (h2 + h1) & _MASK64

    h1 = _fmix64(h1)
    h2 = _fmix64(h2)

    h1 = (h1 + h2) & _MASK64

    # Return upper 32 bits as signed 32-bit int (matching Java's >>> 32)
    return _to_signed64(h1) >> 32  # arithmetic right shift, matching Java


# ============================================================================
# Topology-aware client
# ============================================================================

class CacheTopology:
    """Per-cache topology and segment ownership table."""

    def __init__(self, response):
        self.topology_id = response.topology_id
        self.members = [(m.host, m.port) for m in response.members]
        self.num_segments = len(response.segments)
        # segment_size = ceil(2^31 / num_segments)
        self.segment_size = math.ceil((1 << 31) / self.num_segments)
        # segment_owners[i] = list of (host, port) tuples, primary owner first
        self.segment_owners = []
        for seg in response.segments:
            owners = [self.members[idx] for idx in seg.owner_indices]
            self.segment_owners.append(owners)

    def get_segment(self, key_bytes: bytes) -> int:
        h = murmur3_x64_32(key_bytes)
        normalized = h & 0x7FFFFFFF  # mask sign bit
        return normalized // self.segment_size

    def primary_owner(self, key_bytes: bytes):
        segment = self.get_segment(key_bytes)
        return self.segment_owners[segment][0]


class TopologyAwareClient:
    """
    Minimal topology/hash-aware Infinispan gRPC client.

    Maintains per-member channels and routes key-based operations
    to the primary owner of the key's segment.
    """

    def __init__(self, seed_addresses, cache_name):
        self.cache_name = cache_name
        self.topology = None  # CacheTopology, populated on first connect
        self.channels = {}    # (host, port) -> grpc.Channel
        self.stubs = {}       # (host, port) -> CacheServiceStub

        # Connect to the first reachable seed address
        for host, port in seed_addresses:
            try:
                channel = grpc.insecure_channel(f"{host}:{port}")
                self.channels[(host, port)] = channel
                self._bootstrap(channel)
                break
            except grpc.RpcError:
                continue
        else:
            raise ConnectionError("All seed addresses unreachable")

    def _bootstrap(self, channel):
        """Fetch initial topology and open channels to all members."""
        topo_stub = topology_pb2_grpc.TopologyServiceStub(channel)
        response = topo_stub.GetTopology(
            topology_pb2.TopologyRequest(cache_name=self.cache_name)
        )
        self._update_topology(response)

    def _update_topology(self, response):
        """Update the local topology and adjust connections."""
        self.topology = CacheTopology(response)

        # Open channels to new members
        for host, port in self.topology.members:
            if (host, port) not in self.channels:
                ch = grpc.insecure_channel(f"{host}:{port}")
                self.channels[(host, port)] = ch
                self.stubs[(host, port)] = cache_pb2_grpc.CacheServiceStub(ch)

        # Close channels to departed members
        current = set(self.topology.members)
        for addr in list(self.channels):
            if addr not in current:
                self.channels.pop(addr).close()
                self.stubs.pop(addr, None)

    def _stub_for_key(self, key_bytes: bytes):
        """Return the CacheService stub connected to the primary owner."""
        owner = self.topology.primary_owner(key_bytes)
        return self.stubs[owner]

    def _check_topology(self, trailing_metadata):
        """Check response trailers for topology changes."""
        for key, value in trailing_metadata:
            if key == "x-infinispan-topology-id":
                server_topo_id = int(value)
                if server_topo_id != self.topology.topology_id:
                    # Topology changed — refresh from any connected member
                    any_channel = next(iter(self.channels.values()))
                    topo_stub = topology_pb2_grpc.TopologyServiceStub(
                        any_channel
                    )
                    response = topo_stub.GetTopology(
                        topology_pb2.TopologyRequest(
                            cache_name=self.cache_name
                        )
                    )
                    self._update_topology(response)
                break

    def _request_metadata(self):
        """Build gRPC metadata headers for topology-aware requests."""
        metadata = [("x-infinispan-cache", self.cache_name)]
        if self.topology:
            metadata.append((
                "x-infinispan-topology-id",
                str(self.topology.topology_id)
            ))
        return metadata

    def put(self, key: str, value: bytes, lifespan_ms=0, max_idle_ms=0):
        key_bytes = key.encode("utf-8")
        stub = self._stub_for_key(key_bytes)
        request = cache_pb2.PutRequest(
            key=key_bytes,
            value=value,
            lifespan_ms=lifespan_ms,
            max_idle_ms=max_idle_ms,
        )
        response, call = stub.Put.with_call(
            request, metadata=self._request_metadata()
        )
        self._check_topology(call.trailing_metadata())
        return response

    def get(self, key: str):
        key_bytes = key.encode("utf-8")
        stub = self._stub_for_key(key_bytes)
        request = cache_pb2.GetRequest(key=key_bytes)
        response, call = stub.Get.with_call(
            request, metadata=self._request_metadata()
        )
        self._check_topology(call.trailing_metadata())
        return response.value if response.HasField("value") else None


# ============================================================================
# Usage
# ============================================================================

if __name__ == "__main__":
    client = TopologyAwareClient(
        seed_addresses=[("node1.example.com", 11222)],
        cache_name="my-cache",
    )

    # Put — routed directly to the primary owner of "user:1"
    client.put("user:1", b"some-protobuf-bytes")

    # Get — routed to the same primary owner
    value = client.get("user:1")

    # If a node joins/leaves between calls, the client detects the topology
    # change via the x-infinispan-topology-id trailer and refreshes
    # automatically before the next operation.
```

#### Implementation notes for other languages

The key requirements for a topology-aware client in any language are:

1. **MurmurHash3**: port Infinispan's exact implementation (non-standard constants and
   `bmix`). Verify your implementation against the Java version with known test vectors.
2. **Segment calculation**: use `ceil(2^31 / num_segments)` for segment size, integer
   division for segment ID, and `& 0x7FFFFFFF` to normalize the hash.
3. **gRPC interceptor**: implement a client interceptor that reads
   `x-infinispan-topology-id` from trailing metadata on every response and triggers
   a `GetTopology` call on mismatch. This keeps topology tracking transparent to
   application code.
4. **Channel pool**: maintain one `grpc.Channel` (HTTP/2 connection) per cluster member.
   gRPC multiplexes RPCs over a single connection, so one channel per member is sufficient.
5. **Fallback**: if the primary owner is unreachable, fall back to any other member.
   The server will internally forward the request to the correct owner.

## Server-Initiated Site Switching

### Current state: client-driven site failover

Today, cross-site failover is entirely client-driven. The Hot Rod client pre-configures
backup clusters at startup:

```java
clientBuilder.addServer().host("primary-host").port(port)           // primary cluster
clientBuilder.addCluster("SITE_B").addClusterNode("backup-host", port)  // backup cluster
```

Failover is triggered in two ways:

1. **Automatic**: `OperationDispatcher.trySwitchCluster()` fires when all servers in the
   current cluster are unreachable. It pings each candidate cluster's initial servers in
   order and switches to the first one that responds.
2. **Manual**: `RemoteCacheManager.switchToCluster(clusterName)` allows programmatic
   switching (also exposed via JMX).

The server plays no role in either path — it never tells the client about alternative
sites, their addresses, or their health status.

### What the server knows about remote sites

| Information | Available? | Where |
|-------------|-----------|-------|
| Remote site names | Yes | `BackupConfiguration.site()` — per-cache backup config |
| Site online/offline status | Yes | `TakeOfflineManager` / `XSiteAdminOperations.clusterStatus()` |
| Aggregate cross-cache status | Yes | `GlobalXSiteAdminOperations.globalStatus()` |
| JGroups RELAY2 transport addresses | Yes | `JGroupsTransport` — used for inter-site replication |
| Local site name | Yes | `JGroupsTransport.localSiteName()` (from RELAY2 config) |
| **Client-facing addresses of remote sites** | **No** | Not tracked anywhere — RELAY2 is a cluster-to-cluster transport, not a client protocol |

The critical gap is that the server knows site *names* and *health* but not the
client-connectable addresses of remote site servers.

### Proposed design for gRPC

gRPC can close this gap with two mechanisms:

1. **Site discovery**: the server provides clients with the list of known sites and their
   status, plus optionally the client-facing addresses for each site (if configured).
2. **Server-initiated site switch**: the server can push a directive to connected clients
   telling them to switch to a different site.

#### Prerequisite: server-side site address configuration

For the server to direct clients to a remote site, it must know that site's
client-facing addresses. This requires a new piece of server configuration — a mapping
from site name to client-connectable endpoints:

```xml
<sites local="LON">
  <site name="NYC">
    <client-endpoints>
      <endpoint host="nyc-node1.example.com" port="11222"/>
      <endpoint host="nyc-node2.example.com" port="11222"/>
    </client-endpoints>
  </site>
</sites>
```

This is a server-level (not cache-level) configuration. It is optional — if not
configured, the server can still report site names and status, but cannot provide
addresses for client-directed switching.

As an alternative to static configuration, the server could **request** client-facing
addresses from the remote site via RELAY2 command exchange. Each server knows its own
client-facing address (from its endpoint configuration), so a remote site could respond
with its members' addresses. This would be a new inter-site protocol extension but would
eliminate the static configuration requirement.

#### Site information RPC

```protobuf
message GetSitesRequest {
  // Empty — returns information about all known sites.
}

message SiteInfo {
  string site_name = 1;
  bool is_local = 2;
  SiteStatus status = 3;

  // Client-connectable addresses for this site.
  // Empty if the server doesn't have this information.
  repeated ServerAddress addresses = 4;
}

enum SiteStatus {
  SITE_ONLINE = 0;
  SITE_OFFLINE = 1;
  SITE_MIXED = 2;      // some caches online, some offline to this site
}

message ServerAddress {
  string host = 1;
  int32 port = 2;
}

message GetSitesResponse {
  string local_site = 1;
  repeated SiteInfo sites = 2;
}

service SiteService {
  // Returns information about all known sites.
  rpc GetSites(GetSitesRequest) returns (GetSitesResponse);
}
```

#### Server-initiated site switch

There are two approaches, and they complement each other:

**Approach A: Topology-based switching**

The simplest approach, as suggested: when the server wants clients to move to another
site, it sends a topology update containing the other site's server addresses. From the
client's perspective, this looks like a normal topology change — the server list simply
changes to point at different hosts. The client reconnects to those hosts transparently.

This works within the existing topology machinery (the `x-infinispan-topology-id`
trailer and `GetTopology` / `SubscribeTopology` RPCs described in the topology section).
The server bumps the topology ID and returns a server list consisting of the target
site's addresses.

Pros:
- Zero client-side awareness of "sites" — it's just a topology change
- Works with any client that implements topology awareness
- Immediate effect — next RPC picks up the new topology

Cons:
- The client loses knowledge of the original site's addresses — it can't switch back
  without server help or pre-configuration
- Conflates two different concepts (intra-site topology changes vs. inter-site
  migration), making debugging and monitoring harder
- No client consent — the switch is forced

**Approach B: Explicit site switch directive**

The server sends an explicit site-switch signal via trailers or a dedicated push
mechanism, giving the client the target site name and addresses while preserving
knowledge of all sites:

```protobuf
// Sent as a server-streaming event on the SubscribeTopology stream,
// or as a dedicated notification on a site-subscription stream.

message SiteSwitchDirective {
  // The site the server is directing the client to connect to.
  string target_site = 1;

  // Addresses of the target site's servers.
  repeated ServerAddress addresses = 2;

  // Reason for the switch — informational, for logging/monitoring.
  SiteSwitchReason reason = 3;

  // If true, the server will stop accepting requests after a grace
  // period. The client should switch promptly.
  bool mandatory = 4;
}

enum SiteSwitchReason {
  MAINTENANCE = 0;         // planned maintenance window
  UPGRADE = 1;             // rolling upgrade in progress
  OVERLOADED = 2;          // site is overloaded, shedding load
  ADMIN_INITIATED = 3;     // manual admin action
  SITE_GOING_OFFLINE = 4;  // site is being taken offline
}
```

Pros:
- The client retains full knowledge of all sites and can switch back
- The reason is visible for logging and observability
- The client can implement policies (e.g., respect `mandatory`, ignore `OVERLOADED`
  if it has no alternative)
- Clean separation from topology updates

Cons:
- Requires client-side site-switching logic (but this is desirable for a gRPC client)
- Slightly more complex protocol

**Recommendation: both approaches, layered**

- **Approach B** is the primary mechanism — clients that understand sites get a clean,
  explicit signal with full context.
- **Approach A** can serve as a fallback for simple clients that only implement topology
  awareness but not site awareness. The server can be configured to send a topology
  update as a site switch for clients that don't subscribe to site directives.

#### Delivery mechanism

The site switch directive can be delivered via:

1. **`SubscribeTopology` stream**: extend the existing topology subscription to include
   `SiteSwitchDirective` messages alongside `TopologyUpdate` messages. This is the
   simplest integration since topology-aware clients already have this stream open.

2. **Dedicated `SubscribeSites` stream**: a separate server-streaming RPC for site
   events. Cleaner separation but requires clients to maintain an additional stream.

3. **Trailer on any RPC**: include a `x-infinispan-switch-site` trailer on a normal
   response, similar to how topology changes are signalled. Simple but easy to miss if
   the client is idle.

**Recommendation**: deliver via the `SubscribeTopology` stream by making the response
a `oneof` of topology update and site switch directive:

```protobuf
message TopologyEvent {
  oneof event {
    TopologyUpdate topology_update = 1;
    SiteSwitchDirective site_switch = 2;
  }
}

service TopologyService {
  rpc GetTopology(GetTopologyRequest) returns (GetTopologyResponse);
  rpc SubscribeTopology(SubscribeTopologyRequest) returns (stream TopologyEvent);
  rpc GetSites(GetSitesRequest) returns (GetSitesResponse);
}
```

#### Server-side trigger

The admin triggers a site switch via:

1. **REST API**: `POST /v2/x-site/switch?site=NYC` — sends a `SiteSwitchDirective` to
   all connected gRPC clients with active topology subscriptions.
2. **CLI / JMX**: equivalent admin operation.
3. **Automatic**: the server could automatically send a switch directive when it detects
   that the local site is being taken offline (`GlobalXSiteAdminOperations.takeSiteOffline`).

The server-side implementation iterates over all active `SubscribeTopology` streams
and writes a `SiteSwitchDirective` message to each.

#### Client-side behaviour

On receiving a `SiteSwitchDirective`:

1. Log the directive (site name, reason, mandatory flag).
2. If `mandatory = true`, initiate connection to the target site's addresses within a
   grace period (e.g., 30 seconds).
3. If `mandatory = false`, the client may choose to switch or stay based on its own
   policy (e.g., only switch if current site becomes unhealthy).
4. The client retains knowledge of all sites (from `GetSites` and from received
   directives) and can switch back when the original site is available again.
5. After connecting to the new site, the client opens a new `SubscribeTopology` stream
   and performs normal topology discovery.

#### Graceful drain sequence

For planned maintenance, the full sequence would be:

1. Admin calls `POST /v2/x-site/switch?site=NYC&reason=MAINTENANCE&mandatory=true`
2. Server sends `SiteSwitchDirective(target_site=NYC, mandatory=true, reason=MAINTENANCE)`
   to all subscribed gRPC clients.
3. Clients begin connecting to NYC servers and draining in-flight requests.
4. Server waits for a configurable grace period (or until all gRPC connections close).
5. Admin proceeds with maintenance on the LON site.
6. After maintenance, admin can send a reverse directive from NYC to bring clients back.

### Comparison with Hot Rod site failover

| Aspect | Hot Rod | gRPC (proposed) |
|--------|---------|-----------------|
| **Site discovery** | Client pre-configured | `GetSites` RPC — server provides site list, status, and addresses |
| **Failover trigger** | Client detects all servers dead | Same + server-initiated `SiteSwitchDirective` |
| **Switch mechanism** | Client pings backup clusters in order | Client connects to addresses provided in directive |
| **Server involvement** | None — fully client-driven | Server provides sites, status, and can initiate switch |
| **Switch-back** | `switchToDefaultCluster()` API | Client retains all site info, can switch back anytime |
| **Reason visibility** | None | `SiteSwitchReason` enum — logged and observable |
| **Mandatory switch** | Not possible — server can't tell client to move | `mandatory` flag in directive |
| **Graceful drain** | Not possible | Directive + grace period + connection drain |

### Prerequisites and dependencies

1. **Server-side site address configuration or discovery**: the server must know the
   client-facing addresses of remote sites. This is a new configuration element or a
   new RELAY2 protocol extension.
2. **Topology subscription**: the `SubscribeTopology` stream (from the topology section)
   must be implemented first, as it is the delivery mechanism for site switch directives.
3. **`GlobalXSiteAdminOperations`**: already provides `globalStatus()` — the `GetSites`
   RPC wraps this with address information.

