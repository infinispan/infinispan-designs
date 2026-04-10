# gRPC Server Reflection

## Background

gRPC Server Reflection is the standard mechanism for gRPC servers to expose their service
definitions at runtime — analogous to OpenAPI/Swagger endpoints for REST APIs. It is defined
in [`grpc/reflection/v1/reflection.proto`](https://github.com/grpc/grpc/blob/master/src/proto/grpc/reflection/v1/reflection.proto)
and supported by all major gRPC implementations.

The server exposes a `ServerReflection` service that clients can query to discover:

- All registered services and their methods
- Full message schemas (field names, types, nesting, options)
- File-level proto definitions (as serialized `FileDescriptorProto` blobs)

This enables interactive tooling, dynamic clients, and schema-driven development without
requiring pre-compiled stubs or out-of-band schema distribution.

## Why reflection matters

### Developer tooling

The following tools rely on reflection for service discovery and interactive exploration:

| Tool       | Description                                          |
|------------|------------------------------------------------------|
| `grpcurl`  | Command-line gRPC client (like `curl` for gRPC)     |
| `grpc_cli` | Official gRPC CLI tool                               |
| Postman    | API testing platform (gRPC support via reflection)   |
| Evans      | Interactive gRPC client (REPL-style)                 |
| BloomRPC   | GUI client for gRPC (archived, superseded by Postman)|
| kreya      | GUI gRPC/REST client with reflection support         |

Without reflection, these tools require the `.proto` files to be provided manually. With
reflection, a developer can point any of these tools at the Infinispan server and
immediately explore the API:

```bash
# List all services
$ grpcurl -plaintext localhost:11222 list
org.infinispan.grpc.v1.CacheService
org.infinispan.grpc.v1.SchemaService
grpc.reflection.v1.ServerReflection

# Describe a service
$ grpcurl -plaintext localhost:11222 describe org.infinispan.grpc.v1.CacheService
org.infinispan.grpc.v1.CacheService is a service:
service CacheService {
  rpc Clear ( .org.infinispan.grpc.v1.ClearRequest ) returns ( .org.infinispan.grpc.v1.ClearResponse );
  rpc Get ( .org.infinispan.grpc.v1.GetRequest ) returns ( .org.infinispan.grpc.v1.GetResponse );
  rpc Put ( .org.infinispan.grpc.v1.PutRequest ) returns ( .org.infinispan.grpc.v1.PutResponse );
  rpc Query ( .org.infinispan.grpc.v1.QueryRequest ) returns ( .org.infinispan.grpc.v1.QueryResponse );
  ...
}

# Describe a message type
$ grpcurl -plaintext localhost:11222 describe org.infinispan.grpc.v1.PutRequest
org.infinispan.grpc.v1.PutRequest is a message:
message PutRequest {
  bytes key = 1;
  bytes value = 2;
  int64 lifespan_ms = 3;
  int64 max_idle_ms = 4;
  int32 flags = 5;
}

# Invoke an RPC interactively
$ grpcurl -plaintext \
    -H 'x-infinispan-cache: myCache' \
    -H 'x-infinispan-value-type: sample_bank_account.User' \
    -d '{"key": "dXNlci0xMjM=", "value": "CgRKb2huEBw="}' \
    localhost:11222 org.infinispan.grpc.v1.CacheService/Put
{}
```

### Dynamic clients

Reflection enables building gRPC clients that discover the API at runtime without
pre-compiled stubs. This is useful for:

- **Admin tooling**: a generic Infinispan admin CLI that discovers available operations
  from the server itself
- **Monitoring and observability**: tools that inspect available services and call health
  or metrics endpoints dynamically
- **Gateway/proxy integration**: API gateways (Envoy, Kong, etc.) that route based on
  service discovery via reflection

### Schema distribution

Reflection eliminates the need to distribute `.proto` files out-of-band. Clients can
obtain the complete schema from the running server. This is particularly valuable for
Infinispan because:

- User-registered protobuf schemas (stored in `___protobuf_metadata`) are dynamic — they
  can be added or updated at runtime via the REST API or programmatically
- The server can expose both its own service schemas (CacheService, SchemaService) AND
  user-registered entity schemas through a single reflection endpoint

## The reflection protocol

### Service definition

```protobuf
// From grpc/reflection/v1/reflection.proto (standard, not Infinispan-specific)

syntax = "proto3";

package grpc.reflection.v1;

service ServerReflection {
  // Bidirectional streaming: client sends requests, server responds with
  // file descriptors, service lists, or error messages.
  rpc ServerReflectionInfo(stream ServerReflectionRequest)
      returns (stream ServerReflectionResponse);
}

message ServerReflectionRequest {
  string host = 1;
  oneof message_request {
    string file_by_filename = 3;             // get FileDescriptorProto by .proto filename
    string file_containing_symbol = 4;       // get FileDescriptorProto containing a symbol
    ExtensionRequest file_containing_extension = 5;
    string list_services = 7;                // list all services (value ignored)
    string all_extension_numbers_of_type = 6;
  }
}

message ServerReflectionResponse {
  string valid_host = 1;
  ServerReflectionRequest original_request = 2;
  oneof message_response {
    FileDescriptorResponse file_descriptor_response = 4;
    ExtensionNumberResponse all_extension_numbers_response = 5;
    ListServiceResponse list_services_response = 6;
    ErrorResponse error_response = 7;
  }
}

message FileDescriptorResponse {
  // Serialized google.protobuf.FileDescriptorProto messages.
  // The file descriptor for the requested file plus all transitively
  // imported files.
  repeated bytes file_descriptor_proto = 1;
}

message ListServiceResponse {
  repeated ServiceResponse service = 1;
}

message ServiceResponse {
  string name = 1;  // fully-qualified service name
}
```

### Wire format

The key payload is `FileDescriptorResponse.file_descriptor_proto` — each element is a
serialized `google.protobuf.FileDescriptorProto` message. This is the standard protobuf
self-description format: it encodes the complete `.proto` file structure including all
messages, enums, services, methods, options, and dependencies.

Clients parse these bytes using `google.protobuf.FileDescriptorProto` (available in every
language's protobuf library) to reconstruct the full schema locally.

## Implementation with Strategy C

Since we implement the gRPC wire protocol natively (no grpc-java), we implement the
reflection service ourselves. This is straightforward because ProtoStream already has
the necessary schema model.

### ProtoStream descriptor model mapping

ProtoStream's `FileDescriptor` and related classes map directly to the fields in
`google.protobuf.FileDescriptorProto`:

| ProtoStream class         | FileDescriptorProto field      | Notes                         |
|---------------------------|--------------------------------|-------------------------------|
| `FileDescriptor`          | `FileDescriptorProto`          | Top-level: name, package, dependencies |
| `Descriptor`              | `DescriptorProto`              | Message definitions           |
| `FieldDescriptor`         | `FieldDescriptorProto`         | Field name, number, type, label |
| `EnumDescriptor`          | `EnumDescriptorProto`          | Enum definitions              |
| `EnumValueDescriptor`     | `EnumValueDescriptorProto`     | Enum value name + number      |
| `OneOfDescriptor`         | `OneofDescriptorProto`         | Oneof group definitions       |
| `MapDescriptor`           | (map entry DescriptorProto)    | Synthetic map entry messages  |
| `ServiceDescriptor` (new) | `ServiceDescriptorProto`       | Service definitions (requires ProtoStream enhancement) |
| `MethodDescriptor` (new)  | `MethodDescriptorProto`        | Method name, input/output type, streaming flags |

The `ServiceDescriptor` and `MethodDescriptor` classes are the same ProtoStream
enhancements required by Strategy C for gRPC dispatch routing. Reflection reuses them
for schema exposition — no additional model work needed.

### Serializing to FileDescriptorProto format

The server must serialize ProtoStream's `FileDescriptor` objects into
`google.protobuf.FileDescriptorProto` binary format. This is a one-time serialization
per schema (cacheable), not per-request:

```java
public class FileDescriptorProtoSerializer {

   /**
    * Serializes a ProtoStream FileDescriptor into the standard
    * google.protobuf.FileDescriptorProto binary format.
    *
    * This produces the same bytes that protoc --descriptor_set_out generates.
    */
   static byte[] serialize(FileDescriptor fd) {
      // FileDescriptorProto field numbers (from descriptor.proto):
      //   1: name (string)
      //   2: package (string)
      //   3: dependency (repeated string)
      //   4: message_type (repeated DescriptorProto)
      //   5: enum_type (repeated EnumDescriptorProto)
      //   6: service (repeated ServiceDescriptorProto)
      //   8: options (FileOptions)
      //  12: syntax (string)

      ByteArrayOutputStream out = new ByteArrayOutputStream();
      TagWriter w = TagWriterImpl.newInstance(out);

      w.writeString(1, fd.getName());
      if (fd.getPackage() != null) {
         w.writeString(2, fd.getPackage());
      }
      for (String dep : fd.getDependencies()) {
         w.writeString(3, dep);
      }
      for (Descriptor msg : fd.getMessageTypes()) {
         w.writeBytes(4, serializeDescriptor(msg));
      }
      for (EnumDescriptor en : fd.getEnumTypes()) {
         w.writeBytes(5, serializeEnumDescriptor(en));
      }
      for (ServiceDescriptor svc : fd.getServices()) {
         w.writeBytes(6, serializeServiceDescriptor(svc));
      }
      w.writeString(12, "proto3");
      w.flush();
      return out.toByteArray();
   }
}
```

Since ProtoStream uses the same tag/wire-type encoding as standard protobuf, the
serializer writes directly to the `google.protobuf.FileDescriptorProto` format using
ProtoStream's own `TagWriter`. No `protobuf-java` dependency is needed.

### What schemas to expose

The reflection service should expose three categories of schemas:

| Category | Source | Examples |
|----------|--------|---------|
| **gRPC service schemas** | Compiled into the server | `cache-operations.proto` (CacheService, SchemaService) |
| **User entity schemas** | Registered at runtime via `___protobuf_metadata` cache | `sample_bank_account.proto` (User, Transaction) |
| **ProtoStream built-in schemas** | ProtoStream library | `message-wrapping.proto` (WrappedMessage) |

The server maintains a registry of `FileDescriptor` objects from all three sources. On a
reflection request, it finds the relevant `FileDescriptor` (by filename or by symbol
lookup) and serializes it plus all transitive dependencies.

### Exposing type IDs via custom options

Infinispan's ProtoStream type IDs can be exposed through protobuf custom options. This
makes type IDs discoverable through standard reflection tooling:

```protobuf
// infinispan_options.proto — custom options for Infinispan metadata
syntax = "proto3";

package org.infinispan;

import "google/protobuf/descriptor.proto";

extend google.protobuf.MessageOptions {
  optional int32 type_id = 50001;  // ProtoStream type ID
}
```

User schemas would then include these options:

```protobuf
import "org/infinispan/infinispan_options.proto";

message User {
  option (org.infinispan.type_id) = 42;

  string name = 1;
  int32 age = 2;
  string email = 3;
}
```

A reflection-aware client can parse the `FileDescriptorProto`, read the custom option on
each message, and build the `type_name → type_id` mapping without calling
`SchemaService.GetTypes`. This provides an alternative discovery path through standard
protobuf tooling.

The `SchemaService.GetTypes` RPC remains useful as a lightweight, purpose-built
alternative that returns just the name → ID map without requiring the client to parse
`FileDescriptorProto` and extract custom options.

### Implementation classes

| Class | Module | Purpose |
|-------|--------|---------|
| `GrpcReflectionService` | `server/grpc` | Implements `grpc.reflection.v1.ServerReflection` |
| `FileDescriptorProtoSerializer` | `server/grpc` | Serializes ProtoStream `FileDescriptor` → `FileDescriptorProto` bytes |
| `ReflectionRegistry` | `server/grpc` | Indexes file descriptors by filename and symbol for lookup |
| `ProtobufMetadataManagerImpl` | `server/core` | Provides user-registered schemas (existing class) |

### Caching

`FileDescriptorProto` serialization is deterministic and cacheable. The server serializes
each `FileDescriptor` once (on registration or first request) and caches the result.
Schema registration changes (rare) invalidate the cache. The reflection service itself
is stateless — it reads from the cache on each request.

## Relationship to `SchemaService.GetTypes`

Both reflection and `GetTypes` provide type introspection, but they serve different
purposes:

| Aspect | Server Reflection | SchemaService.GetTypes |
|--------|-------------------|------------------------|
| **Standard** | Yes (grpc.reflection.v1) | Infinispan-specific |
| **What it returns** | Full proto schema (all messages, fields, services, options) | Just `map<string, int32>` name → ID |
| **Client effort** | Parse FileDescriptorProto, extract custom options | Ready-to-use map |
| **Use case** | Tooling, dynamic clients, schema exploration | Optimized encoding tier (WrappedMessage wrapping) |
| **Type IDs** | Via custom options (if present) | Directly in response |
| **Performance** | Heavier (full schema transfer) | Lightweight (one map) |

Both should be implemented. `GetTypes` is the fast path for client SDKs that just need
type IDs. Reflection is the standard path for tooling and schema discovery.

## Scope and phasing

**v1**: implement `grpc.reflection.v1.ServerReflection` with support for:
- `list_services` — enumerate all registered gRPC services
- `file_containing_symbol` — find the proto file containing a service or message
- `file_by_filename` — retrieve a specific proto file by name

This covers the needs of `grpcurl`, Postman, and similar tooling. Extension-related
requests (`file_containing_extension`, `all_extension_numbers_of_type`) can return
`NOT_FOUND` initially — they are rarely used in practice.

**Future**: expose user-registered entity schemas through the same reflection endpoint,
enabling tools to inspect cache value types interactively. This requires merging the
gRPC service `FileDescriptor` set with the user schemas from `___protobuf_metadata`.
