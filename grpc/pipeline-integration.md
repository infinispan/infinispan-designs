# Pipeline Integration and Implementation Strategy
## Background: Single-Port Protocol Detection

Infinispan Server exposes a single TCP port (default 11222) that multiplexes several protocols.
On a new connection the Netty pipeline contains a chain of `ProtocolDetector` instances (each
extending `ByteToMessageDecoder`) that inspect the first bytes of the incoming stream:

| Detector                  | Trigger                      | Module             |
|---------------------------|------------------------------|--------------------|
| `HotRodDetector`          | Magic byte `0xA0`            | `server/hotrod`    |
| `RespDetector`            | RESP3 `HELLO`/`AUTH` pattern | `server/resp`      |
| `MemcachedTextDetector`   | `"set "` ASCII prefix        | `server/memcached` |
| `MemcachedBinaryDetector` | Memcached binary magic byte  | `server/memcached` |

If none of the detectors match, the connection falls through to the HTTP pipeline (REST).

### Pipeline layout (non-SSL)

```
[iprules] -> [stats] -> [HotRod det.] -> [RESP det.] -> [Memcached det.] ->
[CleartextHttp2ServerUpgradeHandler] -> [Http2FrameCodec] -> [Http2MultiplexHandler]
                                                                     |
                                            per-stream sub-channel:  v
                              [Http2StreamFrameToHttpObjectCodec]
                              [AccessControlFilter]
                              [HttpContentCompressor]
                              [HttpContentDecompressor]
                              [HttpObjectAggregator]
                              [StreamCorrelatorHandler]
                              [CorsHandler]
                              [HttpServerKeepAliveHandler]
                              [ChunkedWriteHandler]
                              [RestRequestHandler]          <-- terminal handler
```

### Pipeline layout (SSL / ALPN)

```
[iprules] -> [stats] -> [sni] -> [HotRod det.] -> [RESP det.] -> [Memcached det.] ->
[ALPNHandler]
     |
     +-- ALPN "h2" / "http/1.1"  -->  configureHttpPipeline()  (same sub-channel layout as above)
     +-- ALPN "HR"               -->  HotRodServer.getInitializer()
     +-- ALPN "RP"               -->  RespServer.getInitializer()
     +-- ALPN "MB"               -->  MemcachedServer.getInitializer()
```

### Key classes and locations

| Class                          | Module          | Path                                                                          |
|--------------------------------|-----------------|-------------------------------------------------------------------------------|
| `ProtocolDetector`             | `server/core`   | `server/core/src/main/java/org/infinispan/server/core/ProtocolDetector.java`  |
| `MagicByteDetector`            | `server/core`   | `server/core/src/main/java/org/infinispan/server/core/MagicByteDetector.java` |
| `ALPNHandler`                  | `server/rest`   | `server/rest/src/main/java/org/infinispan/rest/ALPNHandler.java`              |
| `RestRequestHandler`           | `server/rest`   | `server/rest/src/main/java/org/infinispan/rest/RestRequestHandler.java`       |
| `RestChannelInitializer`       | `server/rest`   | `server/rest/src/main/java/org/infinispan/rest/RestChannelInitializer.java`   |
| `SinglePortEndpointRouter`     | `server/router` | `server/router/.../singleport/SinglePortEndpointRouter.java`                  |
| `SinglePortChannelInitializer` | `server/router` | `server/router/.../singleport/SinglePortChannelInitializer.java`              |
| `AbstractProtocolServer`       | `server/core`   | `server/core/src/main/java/org/infinispan/server/core/AbstractProtocolServer.java` |
| `GrpcServer` (new)             | `server/grpc`   | `server/grpc/src/main/java/.../GrpcServer.java`                               |
| `GrpcChannelInitializer` (new) | `server/grpc`   | `server/grpc/src/main/java/.../GrpcChannelInitializer.java`                    |
| `GrpcRequestHandler` (new)     | `server/grpc`   | `server/grpc/src/main/java/.../GrpcRequestHandler.java`                        |
| `GrpcRoutingHandler` (new)     | `server/grpc`   | `server/grpc/src/main/java/.../GrpcRoutingHandler.java`                        |

## Why gRPC Cannot Use a `ProtocolDetector`

gRPC runs **on top of HTTP/2**. At the TCP byte level, a gRPC connection is indistinguishable from
a plain HTTP/2 connection:

1. **Connection preface** (24 bytes): `PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n` — identical for both.
2. **SETTINGS frame** — no reliable difference.
3. **First HEADERS frame** — **this is where they diverge**.

The earliest reliable distinction point is the `content-type` HTTP header in the first HEADERS
frame. Per the [gRPC over HTTP/2 spec](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md),
every gRPC request **must** include:

| Header         | gRPC value                                                        |
|----------------|-------------------------------------------------------------------|
| `content-type` | `application/grpc` (or `application/grpc+proto`, `+json`, etc.)   |
| `:method`      | `POST`                                                            |
| `te`           | `trailers`                                                        |

The canonical discriminator is **`content-type` starting with `application/grpc`**. This header is
mandatory per spec and no legitimate REST request would use it.

Detection therefore **cannot** happen at the raw-byte / connection-preface level. It must happen
**after** HTTP/2 framing is decoded, by inspecting the first request's headers — i.e. at the
request level inside the per-stream sub-channel pipeline.

## Server class design

The gRPC connector follows the same `AbstractProtocolServer` pattern as all other
Infinispan protocol servers. A `GrpcServer` class extends `AbstractProtocolServer<GrpcServerConfiguration>`
and can run in two modes:

1. **Standalone mode** (`startTransport = true`): the server creates its own `NettyTransport`
   and binds to a dedicated socket. The entire port is gRPC — no content-type detection
   needed.

2. **Single-port mode** (`startTransport = false`): the server does not create a transport.
   Instead, `SinglePortEndpointRouter` calls `setEnclosingProtocolServer(this)` and routes
   gRPC traffic via content-type detection inside the shared HTTP/2 pipeline.

This mirrors the pattern used by `HotRodServer`, `RestServer`, `RespServer`, and
`MemcachedServer`.

### GrpcServer skeleton

```java
package org.infinispan.server.grpc;

public class GrpcServer extends AbstractProtocolServer<GrpcServerConfiguration> {

   public GrpcServer() {
      super("gRPC");
   }

   @Override
   public ChannelInitializer<Channel> getInitializer() {
      // Used in standalone mode for transport binding,
      // and by SinglePortEndpointRouter to build the routing table.
      return new NettyInitializers(
            new GrpcChannelInitializer(this, transport));
   }

   @Override
   public ChannelOutboundHandler getEncoder() { return null; }

   @Override
   public ChannelInboundHandler getDecoder() { return null; }

   @Override
   public ChannelMatcher getChannelMatcher() {
      return channel -> true;
   }

   @Override
   public void installDetector(Channel ch) {
      // gRPC cannot be detected at the byte level — detection happens
      // at the HTTP/2 request level via content-type header.
      // This is a no-op; routing is handled by GrpcRoutingHandler
      // inside the per-stream sub-channel pipeline.
   }

   @Override
   public Protocol getProtocol() {
      return Protocol.GRPC;   // new enum value
   }
}
```

### Configuration

```xml
<!-- Standalone: dedicated socket -->
<endpoints>
  <endpoint socket-binding="default" security-realm="default">
    <hotrod-connector />
    <rest-connector />
  </endpoint>
  <endpoint socket-binding="grpc" security-realm="default">
    <grpc-connector />
  </endpoint>
</endpoints>

<!-- Single-port: shared socket (default) -->
<endpoints>
  <endpoint socket-binding="default" security-realm="default">
    <hotrod-connector />
    <rest-connector />
    <grpc-connector />
  </endpoint>
</endpoints>
```

When inside a single-port endpoint, the `EndpointConfigurationBuilder` sets
`startTransport(false)` on the gRPC configuration (via `applyConfigurationToProtocol`),
and `Server.java` adds a `GrpcServerRouteDestination` to the routing table — following the
same pattern as `HotRodServerRouteDestination`, `RestServerRouteDestination`, etc.

## Standalone mode

### Pipeline layout

In standalone mode, `GrpcServer` owns the entire port. There is no REST fallback and no
content-type detection — every connection is gRPC. The `GrpcChannelInitializer` sets up
the HTTP/2 pipeline directly:

```
connection-level pipeline:

    [SslHandler]                  <-- if TLS enabled
    [AccessControlFilter]         <-- IP filtering
    [Http2FrameCodec]
    [Http2MultiplexHandler]
           |
           |  per-stream sub-channel:
           v
    [Http2StreamFrameToHttpObjectCodec]
    [AccessControlFilter]         <-- per-stream IP filtering
    [HttpObjectAggregator]        <-- assembles full request body (unary RPCs)
    [StreamCorrelatorHandler]     <-- HTTP/2 stream ID propagation
    [GrpcRequestHandler]          <-- gRPC dispatch, auth, framing
```

This is simpler than the single-port pipeline: no protocol detectors, no
`CleartextHttp2ServerUpgradeHandler`, no REST handlers, no CORS, no compression
handlers, no keep-alive, no chunked writes. The `GrpcRequestHandler` is the terminal
handler on every stream.

For cleartext (non-TLS) connections, HTTP/2 prior knowledge (direct connection preface)
is the expected mode — gRPC clients connecting to a dedicated gRPC port will send the
HTTP/2 connection preface directly without HTTP/1.1 upgrade negotiation.

For TLS connections, ALPN negotiation with `h2` is used. Since the entire port is gRPC,
the ALPN handler only needs to accept `h2`:

```java
public class GrpcChannelInitializer extends NettyChannelInitializer<GrpcServer> {

   @Override
   public void initializeChannel(Channel ch) throws Exception {
      super.initializeChannel(ch);  // SSL, IP filtering, stats

      if (server.getConfiguration().ssl().enabled()) {
         ch.pipeline().addLast(new GrpcAlpnHandler(server));
      } else {
         // Cleartext HTTP/2 prior knowledge — no upgrade
         configureGrpcPipeline(ch.pipeline(), server);
      }
   }

   static void configureGrpcPipeline(ChannelPipeline pipeline, GrpcServer server) {
      Http2FrameCodec h2c = Http2FrameCodecBuilder.forServer()
            .initialSettings(Http2Settings.defaultSettings())
            .build();
      Http2MultiplexHandler multiplexHandler = new Http2MultiplexHandler(
            new ChannelInitializer<>() {
               @Override
               protected void initChannel(Channel channel) {
                  ChannelPipeline p = channel.pipeline();
                  p.addLast(new Http2StreamFrameToHttpObjectCodec(true));
                  p.addLast(new AccessControlFilter<>(
                        server.getConfiguration(), false));
                  p.addLast(new HttpObjectAggregator(
                        server.maxContentLength()));
                  p.addLast(new StreamCorrelatorHandler());
                  p.addLast(new GrpcRequestHandler(server));
               }
            });
      pipeline.addLast(h2c);
      pipeline.addLast(multiplexHandler);
   }
}
```

### Advantages of standalone mode

- **Simpler pipeline**: no content-type detection, no REST handler fallback.
- **Dedicated resources**: separate socket-binding allows independent tuning of TCP
  options, TLS configuration, and `SETTINGS_MAX_CONCURRENT_STREAMS`.
- **Isolation**: gRPC traffic does not compete with REST/Hot Rod for pipeline processing
  on the same connection.
- **Firewall-friendly**: operators can expose only the gRPC port to specific clients
  while keeping the main endpoint internal.

## Single-port mode

### Detection point

The gRPC routing handler must sit inside the HTTP/2 per-stream sub-channel pipeline, **after**
`Http2StreamFrameToHttpObjectCodec` (which converts HTTP/2 frames into `FullHttpRequest` objects)
and **before** `RestRequestHandler` (which would otherwise consume the request as REST).

```
per-stream sub-channel pipeline (with gRPC support):

    [Http2StreamFrameToHttpObjectCodec]
    [AccessControlFilter]            <-- shared: IP filtering applies to gRPC
    [HttpContentCompressor]          \
    [HttpContentDecompressor]         |
    [HttpObjectAggregator]            |-- REST-only handlers
    [StreamCorrelatorHandler]         |
    [CorsHandler]                     |
    [HttpServerKeepAliveHandler]     /
    [ChunkedWriteHandler]           /
    [GrpcRequestHandler]            <-- NEW: checks content-type, handles or passes through
    [RestRequestHandler]
```

The `GrpcRequestHandler` sits after the shared handlers and before the REST-specific ones.
When it detects a gRPC request, it **replaces the pipeline** for that stream — removing the
REST handlers above it and installing the gRPC-specific pipeline:

```
per-stream sub-channel pipeline (after gRPC detection):

    [Http2StreamFrameToHttpObjectCodec]
    [AccessControlFilter]            <-- kept: IP filtering
    [HttpObjectAggregator]           <-- kept: assembles full request body (unary RPCs)
    [StreamCorrelatorHandler]        <-- kept: HTTP/2 stream ID propagation
    [GrpcRequestHandler]             <-- handles gRPC dispatch, auth, framing
```

Handlers **skipped** for gRPC and why:

| Handler                      | Reason for skipping                                     |
|------------------------------|---------------------------------------------------------|
| `HttpContentCompressor`      | gRPC uses per-message compression (`grpc-encoding`      |
|                              | header + compressed flag in 5-byte frame prefix);       |
|                              | HTTP-level compression would double-compress or corrupt |
| `HttpContentDecompressor`    | Same — decompression is at the gRPC message level       |
| `CorsHandler`                | CORS is a browser security mechanism; gRPC clients are  |
|                              | native applications, not browsers                       |
| `HttpServerKeepAliveHandler` | HTTP/1.1 concept; HTTP/2 connections are persistent     |
| `ChunkedWriteHandler`        | gRPC writes complete DATA frames, not chunked HTTP      |

On each request the `GrpcRequestHandler` inspects the `content-type` header:

- **Starts with `application/grpc`**: handle as gRPC, do not propagate to `RestRequestHandler`.
- **Otherwise**: call `ctx.fireChannelRead(request)` to pass through to `RestRequestHandler`.

### Required change in `server/rest`

The HTTP/2 sub-channel pipeline is currently constructed inside
`ALPNHandler.configureHttpPipeline()` (line 100-142), which is `private static`. The per-stream
`ChannelInitializer` is an anonymous inner class that calls `addCommonHandlers()` — there is no
hook for external modules to inject handlers.

**Minimal API change**: make `configureHttpPipeline` (or `addCommonHandlers`) accept an optional
`ChannelHandler` parameter that, when non-null, is inserted into the sub-channel pipeline
immediately before `RestRequestHandler`. This keeps the change in `server/rest` small and
backwards-compatible — when no extra handler is provided, behaviour is unchanged.

Possible signature:

```java
static void configureHttpPipeline(ChannelPipeline pipeline, RestServer restServer,
                                  ChannelHandler preRestHandler)
```

The existing `configureHttpPipeline(pipeline, restServer)` overload would delegate with
`preRestHandler = null`.

### Router module (`server/router`)

`SinglePortChannelInitializer` would:

1. Instantiate a `GrpcRoutingHandler` — a `SimpleChannelInboundHandler<FullHttpRequest>` that
   checks `content-type` for the `application/grpc` prefix.
2. Pass it to the new `configureHttpPipeline` overload (or equivalent API).
3. On gRPC match: remove subsequent REST handlers from the sub-channel pipeline and install the
   gRPC handler chain.
4. On non-match: `ctx.fireChannelRead(request)` to fall through to `RestRequestHandler`.

This keeps all gRPC logic wholly in `server/router` (or a new `server/grpc` module), with only
a minimal extensibility hook added to `ALPNHandler`.

### Router infrastructure

The following new classes are needed in `server/router` to wire gRPC into the routing
table, following the pattern of `HotRodServerRouteDestination`, `RespServerRouteDestination`,
etc.:

```java
// Route destination — wraps the GrpcServer reference
package org.infinispan.server.router.routes.grpc;

public class GrpcServerRouteDestination extends RouteDestination<GrpcServer> {
   public GrpcServerRouteDestination(String name, GrpcServer server) {
      super(name, server);
   }
}
```

In `SinglePortEndpointRouter.getInitializer()`, gRPC is added alongside the existing
protocol routes:

```java
routingTable.streamRoutes(SinglePortRouteSource.class,
      GrpcServerRouteDestination.class)
   .findFirst()
   .ifPresent(r -> {
      GrpcServer grpcServer = r.getRouteDestination().getProtocolServer();
      // Pass the GrpcRoutingHandler to the HTTP/2 pipeline builder
      grpcRoutingHandler = new GrpcRoutingHandler(grpcServer);
   });
```

In `Server.java`, the startup logic adds gRPC to the routing table when
`startTransport = false`:

```java
} else if (protocolServer instanceof GrpcServer) {
   routes.add(new Route<>(routeSource,
         new GrpcServerRouteDestination(protocolServerName, (GrpcServer) protocolServer)));
}
```

Note that unlike Hot Rod, RESP, and Memcached, gRPC does **not** use `installDetector()`
for byte-level detection. Its `installDetector()` is a no-op. Detection is purely at the
HTTP request level inside the per-stream sub-channel pipeline via the `GrpcRoutingHandler`.

### Alternative considered: replicate HTTP/2 pipeline in router

`SinglePortChannelInitializer` could avoid calling `ALPNHandler.configurePipeline()` entirely and
build its own HTTP/2 pipeline with the gRPC handler baked in. This was rejected because it
duplicates significant pipeline construction code from `ALPNHandler` and would be fragile to
maintain as the REST pipeline evolves.

## Implementation Strategies

There are three main strategies for implementing the gRPC handler, ranging from heaviest (full
grpc-java) to lightest (native ProtoStream-based). They differ in dependency footprint, control
over the wire format, and integration complexity with Infinispan's existing Netty pipeline.

### Strategy A: grpc-java with grpc-netty

Use the official [grpc-java](https://github.com/grpc/grpc-java) library (`io.grpc:grpc-netty`,
`io.grpc:grpc-protobuf`, `io.grpc:grpc-stub`).

**How it works:**

grpc-java provides `NettyServerBuilder` to bootstrap a gRPC server. Service implementations
extend generated `*ImplBase` stubs. Proto files are compiled via `protobuf-maven-plugin` + `protoc`
to produce Java stubs and service base classes.

**Integration challenge:**

grpc-java insists on managing its own Netty bootstrap — it creates its own `ServerBootstrap`,
`EventLoopGroup`, `Http2FrameCodec`, and `Http2MultiplexHandler`. There is **no supported API**
to plug grpc-java handlers into a pre-existing Netty pipeline. This creates a fundamental conflict
with Infinispan's single-port architecture, where the HTTP/2 pipeline is already set up by
`ALPNHandler`.

Possible workarounds:

1. **Separate port**: Run grpc-java on its own port via `NettyServerBuilder`. Simple but defeats
   the single-port goal.

2. **Share `EventLoopGroup`**: Use `NettyServerBuilder.workerEventLoopGroup()` and
   `bossEventLoopGroup()` to share Infinispan's event loops, but still bind a separate port.

3. **Internal API hacking**: Reach into grpc-java internals (`NettyServerHandler`,
   `GrpcHttp2ConnectionHandler`) and install them manually in Infinispan's pipeline. This is
   fragile, undocumented, and would break across grpc-java version upgrades.

**Dependency footprint:**

```
io.grpc:grpc-netty               (Netty transport — must match Infinispan's Netty 4.1.132.Final)
io.grpc:grpc-protobuf            (protobuf-java message binding)
io.grpc:grpc-stub                (generated stub base classes)
io.grpc:grpc-api                 (core API)
com.google.protobuf:protobuf-java (Google's protobuf runtime — duplicates ProtoStream's role)
```

This pulls in Google's `protobuf-java` runtime alongside Infinispan's ProtoStream, creating a
dual-serialization stack. Cache values would need to be converted between ProtoStream's
marshalling and protobuf-java's `GeneratedMessage` types.

**Proto compilation:**

Requires `protobuf-maven-plugin` with the `protoc` compiler and the `grpc-java` codegen plugin.
Generates `*Grpc.java` service stubs with `*ImplBase` abstract classes.

**Verdict:**

Best fit when **full gRPC ecosystem compatibility** (streaming, interceptors, deadlines, load
balancing, health checks, reflection) is required and single-port multiplexing can be sacrificed
or worked around. Heaviest option, most dependencies, hardest to integrate into the existing
single-port pipeline.

---

### Strategy B: grpc-netty handlers, no grpc-java framework

Use grpc-java's low-level Netty handler classes (`NettyServerHandler`,
`GrpcHttp2ConnectionHandler`) directly, without the full `Server` / `ServerBuilder` machinery.
Service dispatch is hand-written rather than using generated stubs.

**How it works:**

Instead of bootstrapping via `NettyServerBuilder`, extract and manually instantiate the gRPC
Netty handler that grpc-java uses internally:

- `NettyServerHandler` extends Netty's `Http2ConnectionHandler`
- It can theoretically be installed in a Netty pipeline as a regular handler
- Service dispatch would be done by implementing `ServerCallHandler` or routing based on the
  `:path` header (`/package.Service/Method`)

**Integration challenge:**

grpc-java's `NettyServerHandler` is an internal class (`io.grpc.netty.NettyServerHandler`). Its
constructor takes numerous internal types (`ServerTransportListener`, `TransportTracer`, etc.)
that are not part of the public API. Using it directly means coupling to grpc-java internals.

Additionally, `NettyServerHandler` replaces Netty's HTTP/2 stack entirely — it extends
`Http2ConnectionHandler` and manages its own `Http2Connection`. It cannot coexist with
Infinispan's existing `Http2FrameCodec` + `Http2MultiplexHandler` setup.

**Verdict:**

Fragile and unsupported. All the dependency weight of Strategy A with none of the stability
guarantees. Not recommended.

---

### Strategy C: Native implementation over Netty HTTP/2, with ProtoStream

Implement the gRPC wire protocol directly on top of Infinispan's existing Netty HTTP/2 pipeline,
using ProtoStream for protobuf serialization. No grpc-java dependency.

**How it works:**

The gRPC-over-HTTP/2 wire protocol is straightforward and well-specified:

**Request (client -> server):**
```
HEADERS frame:
  :method = POST
  :path = /package.ServiceName/MethodName
  :scheme = http | https
  content-type = application/grpc[+proto]
  te = trailers

DATA frame(s):
  1 byte:  compressed flag (0 or 1)
  4 bytes: message length (big-endian uint32)
  N bytes: protobuf-encoded message
```

**Response (server -> client):**
```
HEADERS frame:
  :status = 200
  content-type = application/grpc[+proto]

DATA frame(s):
  1 byte:  compressed flag
  4 bytes: message length (big-endian uint32)
  N bytes: protobuf-encoded response message

TRAILERS frame:
  grpc-status = 0 (OK) | 1..16 (error codes)
  grpc-message = optional error description
```

The `GrpcRequestHandler` in the per-stream sub-channel pipeline would:

1. **Parse the `:path`** header to extract the service and method name.
2. **Read the DATA frame(s)**: strip the 5-byte gRPC framing prefix (compressed flag + length),
   then deserialize the protobuf payload using ProtoStream.
3. **Dispatch** to the appropriate service method implementation.
4. **Write the response**: serialize the response via ProtoStream, prepend the 5-byte gRPC
   framing prefix, send as DATA frame, then send TRAILERS with `grpc-status`.

**What ProtoStream provides today:**

- Protobuf binary serialization/deserialization of messages (the DATA frame payload).
- Proto file parsing via JavaCC grammar (`ProtoBuf.jj`).
- Annotation-driven marshaller generation (`@ProtoField`, `@ProtoSchema`).
- A full descriptor model: `FileDescriptor`, `Descriptor`, `FieldDescriptor`,
  `EnumDescriptor`, `MapDescriptor`, `OneOfDescriptor`.

**What ProtoStream is missing for gRPC:**

The parser's JavaCC grammar **does** tokenize `service` and `rpc` keywords and syntactically
parse service blocks (see `ProtoBuf.jj` lines 319-327), but the parsed data is **discarded** —
the `Service()` and `Rpc()` rules return `void` and never store anything. The descriptor model
has no `ServiceDescriptor` or `MethodDescriptor` classes. Specifically:

```
// ProtoBuf.jj lines 319-327 — services are parsed but ignored
void Rpc() : {}
{
    <RPC> RpcName() <LPAREN> [ <STREAM> ] NamedType() <RPAREN>
    <RETURNS> <LPAREN> [ <STREAM> ] NamedType() <RPAREN>
    (( <LBRACE> (Option(null) | EmptyStatement() )+ <RBRACE> ) | <SEMI_COLON> )
}

void Service(FileDescriptor.Builder f) : {}
{
    <SERVICE> ServiceName() <LBRACE> (Option(f) | Rpc() | EmptyStatement() )+ <RBRACE>
}
```

**Required ProtoStream enhancements:**

1. **`ServiceDescriptor`**: name, full name, list of methods, options.
2. **`MethodDescriptor`**: name, input type (`Descriptor`), output type (`Descriptor`),
   client-streaming flag, server-streaming flag.
3. **`FileDescriptor` additions**: `getServices()` returning `List<ServiceDescriptor>`.
4. **Parser changes**: modify `Service()` and `Rpc()` rules to build and store
   `ServiceDescriptor`/`MethodDescriptor` instances in the `FileDescriptor.Builder`.

These are contained changes to ProtoStream's parser and descriptor model. The serialization
engine (marshaller generation, `TagReader`/`TagWriter`, etc.) is unaffected — it already handles
the message types that gRPC methods use as input/output.

**gRPC framing layer:**

A small `GrpcFrameCodec` (or equivalent logic in the handler) handles the 5-byte
length-prefixed framing that wraps each protobuf message:

```java
// Reading:  strip 5-byte prefix, delegate to ProtoStream
boolean compressed = buf.readBoolean();
int length = buf.readInt();
byte[] payload = new byte[length];
buf.readBytes(payload);
Object message = ProtobufUtil.fromByteArray(serCtx, payload, messageType);

// Writing:  prepend 5-byte prefix, delegate to ProtoStream
byte[] payload = ProtobufUtil.toByteArray(serCtx, responseMessage);
buf.writeBoolean(false);  // not compressed
buf.writeInt(payload.length);
buf.writeBytes(payload);
```

**Service dispatch:**

Route based on `:path` (`/package.Service/Method`) to a registry of service implementations.
The service registry maps `(serviceName, methodName)` -> handler function. This can be built
from the enhanced `ServiceDescriptor` / `MethodDescriptor` model or simply registered manually.

**What this does NOT support (initially):**

- **Client streaming / bidirectional streaming**: requires incremental frame processing across
  multiple DATA frames on the same stream. Can be added later but the initial implementation
  can focus on unary RPCs.
- **gRPC interceptors**: no grpc-java interceptor chain. Cross-cutting concerns (auth, logging,
  metrics) are handled via Infinispan's existing Netty handlers or custom middleware.
- **gRPC reflection / health / channelz**: these are grpc-java ecosystem services. Can be
  implemented manually if needed.
- **Deadline propagation**: `grpc-timeout` header parsing must be implemented manually.
- **Compression**: the `grpc-encoding` / `grpc-accept-encoding` headers and per-message
  compression flag. Can be deferred.

**Verdict:**

Lightest option. Zero new dependencies. Full control over the wire format and seamless
integration with the existing single-port Netty pipeline. Leverages ProtoStream for
serialization (no dual-serialization stack). Requires contained enhancements to ProtoStream's
parser and descriptor model. Best fit when gRPC is used primarily as a **transport/API layer**
for Infinispan operations rather than as a full gRPC ecosystem participant.

---

### Strategy comparison

| Aspect                        | A: grpc-java              | B: grpc-netty internal    | C: Native + ProtoStream    |
|-------------------------------|---------------------------|---------------------------|----------------------------|
| **New dependencies**          | grpc-*, protobuf-java     | grpc-*, protobuf-java     | None                       |
| **Single-port integration**   | Difficult (separate port) | Fragile (internal APIs)   | Natural                    |
| **Serialization**             | protobuf-java (duplicate) | protobuf-java (duplicate) | ProtoStream (existing)     |
| **Proto compilation**         | protoc + grpc plugin      | protoc + grpc plugin      | ProtoStream annotations    |
| **Streaming RPCs**            | Built-in                  | Built-in                  | Must implement             |
| **Interceptors / middleware** | Built-in                  | Partial                   | Netty handlers             |
| **gRPC ecosystem services**   | Built-in                  | Manual                    | Manual                     |
| **ProtoStream changes**       | None                      | None                      | Service/method descriptors |
| **Maintenance burden**        | Track grpc-java releases  | Track grpc-java internals | Own the protocol layer     |
| **Control over wire format**  | Low                       | Low                       | Full                       |

## Recommendation

**Strategy C (Native + ProtoStream)** is the best fit for Infinispan because:

1. **Single-port integration** is a hard requirement, and grpc-java cannot be plugged into an
   existing Netty HTTP/2 pipeline.
2. **ProtoStream already handles protobuf serialization** — adding a second protobuf runtime
   (Google's `protobuf-java`) creates unnecessary duplication and conversion overhead.
3. The gRPC wire protocol over HTTP/2 is **simple and stable** — the framing is a 5-byte prefix
   per message, and the header conventions are well-specified.
4. The ProtoStream enhancements (service/method descriptors) are **contained and useful** beyond
   gRPC — they complete the proto schema model.
5. Infinispan already owns its protocol implementations (Hot Rod, RESP, Memcached) rather than
   depending on external server frameworks. gRPC fits the same pattern.

The main trade-off is that streaming RPCs and ecosystem features (reflection, health checks)
must be implemented manually, but these can be added incrementally.

## Future: HTTP/3 and QUIC readiness

### Context

HTTP/3 replaces TCP with QUIC (a UDP-based transport) and HPACK with QPACK for header
compression. gRPC-over-HTTP/3 is specified in [gRFC G2](https://github.com/grpc/proposal/blob/master/G2-http3-protocol.md)
(still in review). Netty provides QUIC and HTTP/3 codecs via incubator modules
(`netty-incubator-codec-quic`, `netty-incubator-codec-http3`), currently at `0.0.x`
versions targeting Netty 4.1, with active work to graduate HTTP/3 into **Netty 4.2 mainline**.

grpc-java has **no HTTP/3 support and no published timeline** ([grpc-java #8897](https://github.com/grpc/grpc-java/issues/8897)).
This makes Strategy C (native implementation) the only viable path for HTTP/3
support in the Java ecosystem — a significant advantage of owning the protocol layer.

### What HTTP/3 brings

| Benefit                        | Detail                                                                                                  |
|--------------------------------|---------------------------------------------------------------------------------------------------------|
| **No head-of-line blocking**   | QUIC provides per-stream loss isolation. A dropped packet on one RPC does not stall others,             |
|                                | unlike HTTP/2 over TCP where all streams share one TCP byte stream.                                     |
| **Faster connection setup**    | QUIC combines transport + TLS in a single handshake (1-RTT first connection, 0-RTT on resumption).      |
|                                | HTTP/2 over TCP requires TCP handshake + TLS handshake = 2-3 RTTs.                                      |
| **Connection migration**       | QUIC uses connection IDs rather than (IP, port) tuples. Clients switching networks (e.g. Wi-Fi          |
|                                | to cellular, or container rescheduling) keep the same logical connection without re-establishing state.  |
| **Improved loss recovery**     | QUIC's per-packet acknowledgement and more accurate RTT estimation reduce tail latency under             |
|                                | lossy network conditions.                                                                               |

These benefits are particularly relevant for cross-site (xsite) gRPC communication and
edge/mobile deployments where network conditions are less predictable.

### Architectural impact

The key insight is that our gRPC handler logic operates at the **HTTP semantics level**
(HEADERS, DATA, TRAILERS), not at the transport level. The handler chain is structured as:

```
Transport layer (changes)           Application layer (reused as-is)
─────────────────────────           ────────────────────────────────
HTTP/2: Http2FrameCodec             Http2StreamFrameToHttpObjectCodec
        Http2MultiplexHandler        AccessControlFilter
                                     HttpObjectAggregator
HTTP/3: Http3ServerConnectionHandler StreamCorrelatorHandler
        Http3RequestStreamInitializer GrpcRequestHandler
```

Everything from `Http2StreamFrameToHttpObjectCodec` onwards converts HTTP/2 frames into
standard `HttpRequest`/`HttpContent`/`HttpResponse` objects — and HTTP/3's equivalent codec
(`Http3RequestStreamInboundHandler`) produces the same types. The `GrpcRequestHandler`,
`StreamCorrelatorHandler`, and all service dispatch logic are **transport-agnostic** and
can be reused without modification.

### What changes for HTTP/3

QUIC fundamentally changes the channel model. HTTP/2 over TCP uses a single
`SocketChannel` with multiplexed streams managed by `Http2MultiplexHandler`. QUIC
uses a `QuicChannel` (one per connection) that spawns `QuicStreamChannel` instances
(one per stream) — multiplexing is handled at the transport level, not the codec level.

#### HTTP/3 standalone pipeline

```
connection-level (QuicChannel):

    [Http3ServerConnectionHandler]
           |
           |  per-stream (QuicStreamChannel):
           v
    [Http3RequestStreamInboundHandler]   <-- converts QPACK-decoded headers + DATA
                                              into HttpRequest/HttpContent objects
    [AccessControlFilter]
    [HttpObjectAggregator]
    [StreamCorrelatorHandler]
    [GrpcRequestHandler]                 <-- identical to HTTP/2 pipeline
```

#### HTTP/3 channel initializer

```java
public class GrpcQuicChannelInitializer extends ChannelInitializer<QuicChannel> {

   @Override
   protected void initChannel(QuicChannel ch) {
      ch.pipeline().addLast(new Http3ServerConnectionHandler(
            new ChannelInitializer<QuicStreamChannel>() {
               @Override
               protected void initChannel(QuicStreamChannel streamCh) {
                  // Reuse the same application-level pipeline
                  ChannelPipeline p = streamCh.pipeline();
                  p.addLast(new Http3RequestStreamInboundHandler());
                  p.addLast(new AccessControlFilter<>(
                        server.getConfiguration(), false));
                  p.addLast(new HttpObjectAggregator(
                        server.maxContentLength()));
                  p.addLast(new StreamCorrelatorHandler());
                  p.addLast(new GrpcRequestHandler(server));
               }
            }));
   }
}
```

#### Server bootstrap

QUIC uses `DatagramChannel` (UDP) rather than `ServerSocketChannel` (TCP):

```java
// HTTP/3 dedicated socket bootstrap
QuicSslContext sslContext = QuicSslContextBuilder.forServer(key, cert)
      .applicationProtocols("h3")
      .build();

Bootstrap bs = new Bootstrap();
bs.group(bossGroup)
   .channel(NioDatagramChannel.class)
   .handler(new QuicServerCodecBuilder()
         .sslContext(sslContext)
         .maxIdleTimeout(30, TimeUnit.SECONDS)
         .initialMaxStreamsUnidirectional(3)
         .initialMaxStreamsBidirectional(100)
         .handler(new GrpcQuicChannelInitializer(server))
         .build())
   .bind(port);
```

### Single-port considerations

HTTP/3 cannot share a TCP port with HTTP/2 — it runs over UDP. However, clients
discover HTTP/3 availability via the `Alt-Svc` HTTP header on an HTTP/2 response:

```
Alt-Svc: h3=":11222"
```

This tells clients that the same logical endpoint supports HTTP/3 on the specified
UDP port. The `RestRequestHandler` (or a shared response filter) can inject this
header into HTTP/2 responses when HTTP/3 is enabled. Clients that support HTTP/3
will upgrade on subsequent connections; those that don't will continue using HTTP/2.

This means HTTP/3 support requires:
1. A dedicated UDP socket (standalone mode only — there is no UDP equivalent of the
   single-port TCP router).
2. An `Alt-Svc` header on HTTP/2 responses to advertise HTTP/3 availability.
3. The same `GrpcServer` instance can manage both transports, exposing the same
   service registry and configuration.

### Design recommendations

1. **No HTTP/3 in v1.** The Netty codecs are still incubating and gRFC G2 is not
   ratified. Focus on HTTP/2, which is the production standard for gRPC.

2. **Keep handlers transport-agnostic.** The current design already achieves this —
   `GrpcRequestHandler` and all service dispatch logic work with standard Netty
   `HttpRequest`/`HttpResponse` objects, not HTTP/2-specific frame types.

3. **Isolate transport setup in channel initializers.** The `configureGrpcPipeline()`
   pattern (a static method that builds the application-level handler chain)
   makes it easy to reuse the same handlers from both `GrpcChannelInitializer`
   (HTTP/2) and a future `GrpcQuicChannelInitializer` (HTTP/3):

   ```java
   // Shared application-level pipeline setup
   static void configureGrpcHandlers(ChannelPipeline p, GrpcServer server) {
      p.addLast(new AccessControlFilter<>(server.getConfiguration(), false));
      p.addLast(new HttpObjectAggregator(server.maxContentLength()));
      p.addLast(new StreamCorrelatorHandler());
      p.addLast(new GrpcRequestHandler(server));
   }
   ```

   Both initializers call `configureGrpcHandlers()` after their respective
   transport-specific setup.

4. **Add `Protocol.GRPC_H3`** (or a transport flag on `GrpcServerConfiguration`)
   when HTTP/3 support is added, so the server can bind both TCP and UDP sockets
   from the same `GrpcServer` instance.
