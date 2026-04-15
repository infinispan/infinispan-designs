# Hot Rod Wire Format Schema

## Problem Statement

The Hot Rod binary protocol is currently described in prose and AsciiDoc tables in
`documentation/src/main/asciidoc/topics/hotrod_protocol.adoc`. While readable by
humans, this format has several drawbacks:

- **No code generation**: every language client must hand-write serializers and
  deserializers from the prose specification, a process that is error-prone and
  hard to keep in sync across languages.
- **No machine-readable schema**: tooling cannot validate captures, generate
  test vectors, or perform conformance testing automatically.
- **Ambiguity**: conditional fields, variable-length encodings, and
  version-specific changes are described in natural language, leaving room for
  misinterpretation.

## Evaluated Approaches

| Tool / Standard | Describes existing formats | Read codegen | Write codegen | Language coverage | Complexity |
|---|---|---|---|---|---|
| **Kaitai Struct** | Yes | Yes (14+ langs) | Community/partial | Java, C++, Python, Go, Rust, C#, JS, ... | Low |
| **ASN.1** | Yes (via custom encoding rules) | Yes | Yes | C, Ada, Java (limited) | Very high |
| **DFDL (Apache Daffodil)** | Yes | Yes | Partial | Java | Medium |
| **Cap'n Proto** | No (imposes own format) | Yes | Yes | 10+ langs | Medium |
| **FlatBuffers** | No (imposes own format) | Yes | Yes | 10+ langs | Medium |
| **Protocol Buffers** | No (imposes own format) | Yes | Yes | 10+ langs | Low |

### Recommendation

**Kaitai Struct** is the best fit because:

1. It can describe the *existing* Hot Rod wire format without requiring protocol changes.
2. It generates deserializers in 14+ languages from a single `.ksy` YAML definition.
3. The `.ksy` files are human-readable and can serve as both documentation and
   machine-readable specification.
4. It handles variable-length integers, conditional fields, enums, and nested
   structures --- all features the Hot Rod protocol uses extensively.
5. It has a web IDE ([ide.kaitai.io](https://ide.kaitai.io)) for interactive
   exploration with hex dumps.

**Limitation**: Kaitai Struct is read-oriented. It generates robust deserializers
natively, but serializer generation is community-contributed and uneven.
For serialization, options include:

- Hand-written serializers guided by the `.ksy` schema (formalizes the current approach).
- A code generation template (Jinja2, JavaPoet, etc.) that reads the `.ksy` and
  emits writers.
- Adopting a symmetric format (Cap'n Proto, FlatBuffers) in a future protocol
  version that breaks backward compatibility.

## Kaitai Struct Primer

A `.ksy` file is a YAML document describing a binary format. Key concepts:

```yaml
meta:
  id: format_name          # identifier used in generated code
  endian: be               # big-endian by default

seq:                       # top-level sequence of fields, read in order
  - id: field_name
    type: u1               # unsigned 1-byte integer
  - id: another_field
    type: str
    encoding: UTF-8
    size: 10

enums:
  my_enum:
    0x01: value_a
    0x02: value_b

types:                     # reusable sub-structures
  my_struct:
    seq:
      - id: length
        type: u4
      - id: data
        size: length
```

Built-in types: `u1`, `u2`, `u4`, `u8` (unsigned), `s1`, `s2`, `s4`, `s8`
(signed), `str`, `bytes`. Custom types are defined in the `types` section.

Conditional fields use `if`:

```yaml
- id: optional_field
  type: u4
  if: some_flag == 1
```

Repeated fields use `repeat`:

```yaml
- id: entries
  type: entry
  repeat: expr
  repeat-expr: entry_count
```

## Prototype: Hot Rod Protocol 4.1 Kaitai Schema

The following `.ksy` schema describes the Hot Rod 4.1 wire format.
It covers the request/response headers, fundamental data types (vInt, vLong),
MediaType encoding, topology change headers, and all operations through
protocol version 4.1.

> **Note**: This is a proof-of-concept. A production schema would be split into
> multiple `.ksy` files and include exhaustive test vectors.

### Fundamental Types

Hot Rod uses two variable-length integer encodings that are not built into
Kaitai but can be expressed as custom types:

```yaml
# hotrod_types.ksy
meta:
  id: hotrod_types
  endian: be

types:
  # Variable-length unsigned integer (1-5 bytes).
  # High bit of each byte indicates continuation.
  # Identical to protobuf varint32.
  vint:
    seq:
      - id: groups
        type: vint_group
        repeat: until
        repeat-until: not _.has_more
    instances:
      value:
        value: >-
          groups[0].value
          + (groups.size > 1 ? (groups[1].value << 7) : 0)
          + (groups.size > 2 ? (groups[2].value << 14) : 0)
          + (groups.size > 3 ? (groups[3].value << 21) : 0)
          + (groups.size > 4 ? (groups[4].value << 28) : 0)

  vint_group:
    seq:
      - id: b
        type: u1
    instances:
      has_more:
        value: (b & 0x80) != 0
      value:
        value: b & 0x7F

  # Variable-length unsigned long (1-9 bytes).
  # Same encoding as vint but extended to 64-bit values.
  vlong:
    seq:
      - id: groups
        type: vint_group
        repeat: until
        repeat-until: not _.has_more
    instances:
      value:
        value: >-
          groups[0].value
          + (groups.size > 1 ? (groups[1].value << 7) : 0)
          + (groups.size > 2 ? (groups[2].value << 14) : 0)
          + (groups.size > 3 ? (groups[3].value << 21) : 0)
          + (groups.size > 4 ? (groups[4].value << 28) : 0)
          + (groups.size > 5 ? (groups[5].value << 35) : 0)
          + (groups.size > 6 ? (groups[6].value << 42) : 0)
          + (groups.size > 7 ? (groups[7].value << 49) : 0)
          + (groups.size > 8 ? (groups[8].value << 56) : 0)

  # A length-prefixed byte array: vInt length followed by that many bytes.
  lp_bytes:
    seq:
      - id: length
        type: vint
      - id: data
        size: length.value

  # A length-prefixed UTF-8 string: vInt length followed by UTF-8 bytes.
  lp_string:
    seq:
      - id: length
        type: vint
      - id: data
        type: str
        encoding: UTF-8
        size: length.value
```

### MediaType Encoding (Protocol 2.8+)

```yaml
  # MediaType descriptor used in request headers since protocol 2.8.
  media_type:
    seq:
      - id: type_indicator
        type: u1
        enum: media_type_kind
      - id: predefined_id
        type: vint
        if: type_indicator == media_type_kind::predefined
      - id: custom_string
        type: lp_string
        if: type_indicator == media_type_kind::custom
      - id: param_count
        type: vint
        if: type_indicator != media_type_kind::none
      - id: params
        type: media_type_param
        repeat: expr
        repeat-expr: param_count.value
        if: type_indicator != media_type_kind::none

  media_type_param:
    seq:
      - id: key
        type: lp_string
      - id: value
        type: lp_string
```

### Request Header (Protocol 4.1)

```yaml
  # Full request header for protocol versions >= 2.8 (with MediaType support).
  request_header:
    seq:
      - id: magic
        type: u1
        # Expected: 0xA0
      - id: message_id
        type: vlong
      - id: version
        type: u1
        # 41 for protocol 4.1
      - id: opcode
        type: u1
        enum: request_opcode
      - id: cache_name_length
        type: vint
      - id: cache_name
        type: str
        encoding: UTF-8
        size: cache_name_length.value
      - id: flags
        type: vint
      - id: client_intelligence
        type: u1
        enum: client_intelligence_type
      - id: topology_id
        type: vint
      # MediaType fields added in protocol 2.8
      - id: key_media_type
        type: media_type
      - id: value_media_type
        type: media_type
```

### Response Header

```yaml
  response_header:
    seq:
      - id: magic
        type: u1
        # Expected: 0xA1
      - id: message_id
        type: vlong
      - id: opcode
        type: u1
        enum: response_opcode
      - id: status
        type: u1
        enum: response_status
      - id: topology_change_marker
        type: u1
      - id: topology_change
        type: topology_change_header
        if: topology_change_marker != 0
```

### Topology Change Headers

```yaml
  # Topology-aware topology change (client intelligence 0x02)
  topology_aware_header:
    seq:
      - id: topology_id
        type: vint
      - id: num_servers
        type: vint
      - id: servers
        type: server_address
        repeat: expr
        repeat-expr: num_servers.value

  server_address:
    seq:
      - id: host
        type: lp_string
      - id: port
        type: u2

  # Hash-distribution-aware topology change (client intelligence 0x03, protocol >= 2.0)
  hash_dist_header_v2:
    seq:
      - id: topology_id
        type: vint
      - id: num_servers
        type: vint
      - id: servers
        type: server_address
        repeat: expr
        repeat-expr: num_servers.value
      - id: hash_function_version
        type: u1
      - id: num_segments
        type: vint
      - id: segments
        type: segment_owners
        repeat: expr
        repeat-expr: num_segments.value

  segment_owners:
    seq:
      - id: num_owners
        type: u1
        # 0, 1, or 2
      - id: first_owner_index
        type: vint
        if: num_owners >= 1
      - id: second_owner_index
        type: vint
        if: num_owners >= 2

  # Generic topology change header - dispatches based on client intelligence.
  # In practice, the client knows its own intelligence level and picks
  # the correct sub-type. This type is a simplified representation.
  topology_change_header:
    seq:
      - id: topology_id
        type: vint
      - id: num_servers
        type: vint
      - id: servers
        type: server_address
        repeat: expr
        repeat-expr: num_servers.value
```

### Entry Metadata (Protocol 4.0+)

```yaml
  # Metadata block returned with previous values since protocol 4.0,
  # and with GetWithMetadata/GetStream responses since protocol 1.2.
  entry_metadata:
    seq:
      - id: flag
        type: u1
        # Bitwise OR of: 0x01 = INFINITE_LIFESPAN, 0x02 = INFINITE_MAXIDLE
      - id: created
        type: s8
        if: (flag & 0x01) == 0
      - id: lifespan
        type: vint
        if: (flag & 0x01) == 0
      - id: last_used
        type: s8
        if: (flag & 0x02) == 0
      - id: max_idle
        type: vint
        if: (flag & 0x02) == 0
      - id: entry_version
        type: s8
```

### Operations

```yaml
  # --- Key-only request body (Get, Remove, ContainsKey, GetWithVersion) ---
  key_request:
    seq:
      - id: key
        type: lp_bytes

  # --- Put/Replace/PutIfAbsent request body (protocol >= 2.2 with TimeUnits) ---
  put_request:
    seq:
      - id: key
        type: lp_bytes
      - id: time_units
        type: u1
        # Lifespan in bits 0-3, MaxIdle in bits 4-7
      - id: lifespan
        type: vlong
        if: (time_units & 0x0F) < 0x07
      - id: max_idle
        type: vlong
        if: ((time_units >> 4) & 0x0F) < 0x07
      - id: value
        type: lp_bytes

  # --- ReplaceIfUnmodified request body ---
  replace_if_unmodified_request:
    seq:
      - id: key
        type: lp_bytes
      - id: time_units
        type: u1
      - id: lifespan
        type: vlong
        if: (time_units & 0x0F) < 0x07
      - id: max_idle
        type: vlong
        if: ((time_units >> 4) & 0x0F) < 0x07
      - id: entry_version
        type: s8
      - id: value
        type: lp_bytes

  # --- RemoveIfUnmodified request body ---
  remove_if_unmodified_request:
    seq:
      - id: key
        type: lp_bytes
      - id: entry_version
        type: s8

  # --- Get response (0x04) ---
  get_response_body:
    seq:
      - id: value
        type: lp_bytes
        if: _parent.header.status == response_status::success

  # --- GetWithMetadata response (0x1C) ---
  get_with_metadata_response_body:
    seq:
      - id: metadata
        type: entry_metadata
        if: _parent.header.status == response_status::success
      - id: value
        type: lp_bytes
        if: _parent.header.status == response_status::success

  # --- Put/Remove/Replace response with metadata (protocol 4.0+) ---
  # Returned when status indicates previous value present (0x03 or 0x04).
  previous_value_with_metadata:
    seq:
      - id: metadata
        type: entry_metadata
      - id: value
        type: lp_bytes

  # --- PutAll request ---
  put_all_request:
    seq:
      - id: time_units
        type: u1
      - id: lifespan
        type: vlong
        if: (time_units & 0x0F) < 0x07
      - id: max_idle
        type: vlong
        if: ((time_units >> 4) & 0x0F) < 0x07
      - id: entry_count
        type: vint
      - id: entries
        type: key_value_pair
        repeat: expr
        repeat-expr: entry_count.value

  key_value_pair:
    seq:
      - id: key
        type: lp_bytes
      - id: value
        type: lp_bytes

  # --- GetAll request ---
  get_all_request:
    seq:
      - id: key_count
        type: vint
      - id: keys
        type: lp_bytes
        repeat: expr
        repeat-expr: key_count.value

  # --- GetAll response ---
  get_all_response_body:
    seq:
      - id: entry_count
        type: vint
      - id: entries
        type: key_value_pair
        repeat: expr
        repeat-expr: entry_count.value

  # --- Stats response ---
  stats_response_body:
    seq:
      - id: num_stats
        type: vint
      - id: stats
        type: stat_entry
        repeat: expr
        repeat-expr: num_stats.value

  stat_entry:
    seq:
      - id: name
        type: lp_string
      - id: value
        type: lp_string

  # --- Query request (0x1F) ---
  query_request:
    seq:
      - id: query_payload
        type: lp_bytes

  # --- Query response (0x20) ---
  query_response_body:
    seq:
      - id: response_payload
        type: lp_bytes

  # --- Size response (0x2A) ---
  size_response_body:
    seq:
      - id: size
        type: vint

  # --- Exec request (0x2B) ---
  exec_request:
    seq:
      - id: script_name
        type: lp_string
      - id: param_count
        type: vint
      - id: params
        type: exec_param
        repeat: expr
        repeat-expr: param_count.value

  exec_param:
    seq:
      - id: name
        type: lp_string
      - id: value
        type: lp_bytes

  # --- Auth mech list response (0x22) ---
  auth_mech_list_response:
    seq:
      - id: mech_count
        type: vint
      - id: mechs
        type: lp_string
        repeat: expr
        repeat-expr: mech_count.value

  # --- Auth request (0x23) ---
  auth_request:
    seq:
      - id: mech
        type: lp_string
      - id: response_data
        type: lp_bytes

  # --- Auth response (0x24) ---
  auth_response:
    seq:
      - id: completed
        type: u1
      - id: challenge_data
        type: lp_bytes

  # --- Iteration Start request (0x31, protocol >= 2.5) ---
  iteration_start_request:
    seq:
      - id: segments_size
        type: vint
        # Signed: -1 means no segment filtering
      - id: segments
        size: segments_size.value
        if: segments_size.value > 0
      - id: filter_converter_size
        type: vint
        # Signed: -1 means no filter
      - id: filter_converter
        type: str
        encoding: UTF-8
        size: filter_converter_size.value
        if: filter_converter_size.value > 0
      - id: param_count
        type: u1
        if: filter_converter_size.value > 0
      - id: params
        type: lp_bytes
        repeat: expr
        repeat-expr: param_count
        if: filter_converter_size.value > 0 and param_count > 0
      - id: batch_size
        type: vint
      - id: metadata
        type: u1
        # 1 = return metadata, 0 = no metadata

  # --- Iteration Next response (0x34) ---
  iteration_next_response:
    seq:
      - id: finished_segments_size
        type: vint
      - id: finished_segments
        size: finished_segments_size.value
      - id: entry_count
        type: vint
      - id: num_value_projections
        type: vint

  # --- BulkGet request (0x19) ---
  bulk_get_request:
    seq:
      - id: entry_count
        type: vint
        # 0 = return all entries

  # --- GetStreamStart request (0xE9, protocol 4.1) ---
  get_stream_start_request:
    seq:
      - id: key
        type: lp_bytes
      - id: batch_size
        type: vint

  # --- GetStreamStart response (0xE8) ---
  get_stream_start_response:
    seq:
      - id: stream_id
        type: s4
        if: _parent.header.status == response_status::success
      - id: complete
        type: u1
        if: _parent.header.status == response_status::success
      - id: metadata
        type: entry_metadata
        if: _parent.header.status == response_status::success
      - id: value
        type: lp_bytes
        if: _parent.header.status == response_status::success

  # --- GetStreamNext request (0xE7) ---
  get_stream_next_request:
    seq:
      - id: stream_id
        type: s4

  # --- GetStreamNext response (0xE6) ---
  get_stream_next_response:
    seq:
      - id: stream_id
        type: s4
        if: _parent.header.status == response_status::success
      - id: complete
        type: u1
        if: _parent.header.status == response_status::success
      - id: value
        type: lp_bytes
        if: _parent.header.status == response_status::success

  # --- GetStreamEnd request (0xE4) ---
  get_stream_end_request:
    seq:
      - id: stream_id
        type: s4

  # --- PutStreamStart request (0xEF) ---
  put_stream_start_request:
    seq:
      - id: key
        type: lp_bytes
      - id: metadata
        type: entry_metadata
      - id: condition_version
        type: s8
        # 0 = unconditional, -1 = put-if-absent, other = conditional replace

  # --- PutStreamStart response (0xEE) ---
  put_stream_start_response:
    seq:
      - id: stream_id
        type: s4

  # --- PutStreamNext request (0xED) ---
  put_stream_next_request:
    seq:
      - id: stream_id
        type: s4
      - id: complete
        type: u1
      - id: chunk
        type: lp_bytes

  # --- PutStreamEnd request (0xEB) ---
  put_stream_end_request:
    seq:
      - id: stream_id
        type: s4

  # --- Add Client Listener request (0x25, protocol >= 2.6) ---
  add_listener_request:
    seq:
      - id: listener_id
        type: lp_bytes
      - id: include_state
        type: u1
      - id: filter_factory_name
        type: lp_string
      - id: filter_factory_params
        type: listener_factory_params
        if: filter_factory_name.length.value > 0
      - id: converter_factory_name
        type: lp_string
      - id: converter_factory_params
        type: listener_factory_params
        if: converter_factory_name.length.value > 0
      - id: use_raw_data
        type: u1
      - id: event_type_interests
        type: vint
        # Bitmask: 0x01=created, 0x02=modified, 0x04=removed, 0x08=expired

  listener_factory_params:
    seq:
      - id: param_count
        type: u1
      - id: params
        type: lp_bytes
        repeat: expr
        repeat-expr: param_count

  # --- Remove Client Listener request (0x27) ---
  remove_listener_request:
    seq:
      - id: listener_id
        type: lp_bytes

  # --- Bloom Filter Listener request (0x41, protocol 3.1) ---
  bloom_filter_listener_request:
    seq:
      - id: listener_id
        type: lp_bytes
      - id: bloom_filter_bit_size
        type: vint

  # --- Update Bloom Filter request (0x42) ---
  update_bloom_filter_request:
    seq:
      - id: bloom_filter_bits
        type: lp_bytes

  # --- Cache Entry Event (sent by server) ---
  cache_entry_created_event:
    seq:
      - id: listener_id
        type: lp_bytes
      - id: custom_marker
        type: u1
      - id: command_retried
        type: u1
      - id: key
        type: lp_bytes
      - id: version
        type: s8

  cache_entry_modified_event:
    seq:
      - id: listener_id
        type: lp_bytes
      - id: custom_marker
        type: u1
      - id: command_retried
        type: u1
      - id: key
        type: lp_bytes
      - id: version
        type: s8

  cache_entry_removed_event:
    seq:
      - id: listener_id
        type: lp_bytes
      - id: custom_marker
        type: u1
      - id: command_retried
        type: u1
      - id: key
        type: lp_bytes

  custom_event:
    seq:
      - id: listener_id
        type: lp_bytes
      - id: custom_marker
        type: u1
        # 1 = marshalled, 2 = raw binary
      - id: event_data
        type: lp_bytes

  # --- XID (X/Open XA Transaction ID) ---
  xid:
    seq:
      - id: format_id
        type: vint
        # Signed vInt (ZigZag encoded)
      - id: global_tx_id_length
        type: u1
      - id: global_tx_id
        size: global_tx_id_length
      - id: branch_qualifier_length
        type: u1
      - id: branch_qualifier
        size: branch_qualifier_length

  # --- Transaction Prepare request (0x7D, protocol >= 2.9) ---
  prepare_request_v2:
    seq:
      - id: transaction_id
        type: xid
      - id: one_phase_commit
        type: u1
      - id: recoverable
        type: u1
      - id: timeout
        type: s8
      - id: num_keys
        type: vint
      - id: keys
        type: tx_write_entry
        repeat: expr
        repeat-expr: num_keys.value

  tx_write_entry:
    seq:
      - id: key
        type: lp_bytes
      - id: control_byte
        type: u1
        # 0x01 = NOT_READ, 0x02 = NON_EXISTING, 0x04 = REMOVE_OPERATION
      - id: version_read
        type: s8
        if: (control_byte & 0x01) == 0 and (control_byte & 0x02) == 0
      - id: time_units
        type: u1
        if: (control_byte & 0x04) == 0
      - id: lifespan
        type: vlong
        if: (control_byte & 0x04) == 0 and (time_units & 0x0F) < 0x07
      - id: max_idle
        type: vlong
        if: (control_byte & 0x04) == 0 and ((time_units >> 4) & 0x0F) < 0x07
      - id: value
        type: lp_bytes
        if: (control_byte & 0x04) == 0

  # --- Transaction Commit/Rollback request ---
  tx_commit_or_rollback_request:
    seq:
      - id: transaction_id
        type: xid

  # --- Transaction Prepare/Commit/Rollback response ---
  tx_response:
    seq:
      - id: xa_return_code
        type: vint
        # XA_OK=0, XA_RDONLY=3, or XA error codes

  # --- Forget Transaction request (0x79) ---
  forget_tx_request:
    seq:
      - id: transaction_id
        type: xid

  # --- Fetch In-Doubt Transactions response (0x7C) ---
  fetch_in_doubt_response:
    seq:
      - id: num_xids
        type: vint
      - id: xids
        type: xid
        repeat: expr
        repeat-expr: num_xids.value

  # --- Counter Configuration ---
  counter_configuration:
    seq:
      - id: flags
        type: u1
        # Bit 0: 1=WEAK, 0=STRONG
        # Bit 1: 1=BOUNDED, 0=UNBOUNDED
        # Bit 2: 1=PERSISTENT, 0=VOLATILE
      - id: concurrency_level
        type: vint
        if: (flags & 0x01) != 0
      - id: lower_bound
        type: s8
        if: (flags & 0x02) != 0
      - id: upper_bound
        type: s8
        if: (flags & 0x02) != 0
      - id: initial_value
        type: s8

  # --- Counter Create request (0x4B) ---
  counter_create_request:
    seq:
      - id: name
        type: lp_string
      - id: configuration
        type: counter_configuration

  # --- Counter Get/Set/Add/Reset/Remove request ---
  counter_name_request:
    seq:
      - id: name
        type: lp_string

  # --- Counter Get-And-Set request (0x7F, protocol 3.1) ---
  counter_get_and_set_request:
    seq:
      - id: name
        type: lp_string
      - id: value
        type: s8

  # --- Counter value response ---
  counter_value_response:
    seq:
      - id: value
        type: s8

  # --- Ping response (protocol >= 3.0) ---
  ping_response_v3:
    seq:
      - id: key_type
        type: media_type
      - id: value_type
        type: media_type
      - id: server_version
        type: u1
      - id: op_count
        type: vint
      - id: supported_opcodes
        type: u2
        repeat: expr
        repeat-expr: op_count.value
```

### Enums

```yaml
enums:
  request_opcode:
    0x01: put
    0x03: get
    0x05: put_if_absent
    0x07: replace
    0x09: replace_if_unmodified
    0x0B: remove
    0x0D: remove_if_unmodified
    0x0F: contains_key
    0x11: get_with_version
    0x13: clear
    0x15: stats
    0x17: ping
    0x19: bulk_get
    0x1B: get_with_metadata
    0x1D: bulk_get_keys
    0x1F: query
    0x21: auth_mech_list
    0x23: auth
    0x25: add_client_listener
    0x27: remove_client_listener
    0x29: size
    0x2B: exec
    0x2D: put_all
    0x2F: get_all
    0x31: iteration_start
    0x33: iteration_next
    0x35: iteration_end
    0x3B: prepare
    0x3D: commit
    0x3F: rollback
    0x41: add_bloom_filter_listener
    0x42: update_bloom_filter
    0x4B: counter_create
    0x4D: counter_get_value
    0x4F: counter_reset
    0x53: counter_add_and_get
    0x57: counter_is_defined
    0x59: counter_add_listener
    0x5B: counter_remove_listener
    0x5D: counter_remove
    0x5F: counter_get_configuration
    0x67: multimap_get
    0x69: multimap_get_with_metadata
    0x6B: multimap_put
    0x6D: multimap_remove_key
    0x6F: multimap_remove_entry
    0x71: multimap_size
    0x73: multimap_contains_entry
    0x75: multimap_contains_key
    0x77: multimap_contains_value
    0x79: forget_tx
    0x7B: fetch_in_doubt_tx
    0x7D: prepare_v2
    0x7F: counter_get_and_set
    0xE4: get_stream_end
    0xE7: get_stream_next
    0xE9: get_stream_start
    0xEB: put_stream_end
    0xED: put_stream_next
    0xEF: put_stream_start

  response_opcode:
    0x02: put
    0x04: get
    0x06: put_if_absent
    0x08: replace
    0x0A: replace_if_unmodified
    0x0C: remove
    0x0E: remove_if_unmodified
    0x10: contains_key
    0x12: get_with_version
    0x14: clear
    0x16: stats
    0x18: ping
    0x1A: bulk_get
    0x1C: get_with_metadata
    0x1E: bulk_get_keys
    0x20: query
    0x22: auth_mech_list
    0x24: auth
    0x26: add_client_listener
    0x28: remove_client_listener
    0x2A: size
    0x2C: exec
    0x2E: put_all
    0x30: get_all
    0x32: iteration_start
    0x34: iteration_next
    0x36: iteration_end
    0x3C: prepare
    0x3E: commit
    0x40: rollback
    0x42: add_bloom_filter_listener
    0x44: update_bloom_filter
    0x4C: counter_create
    0x4E: counter_get_value
    0x50: error
    0x52: counter_reset
    0x54: counter_add_and_get
    0x58: counter_is_defined
    0x5A: counter_add_listener
    0x5C: counter_remove_listener
    0x5E: counter_remove
    0x60: cache_entry_created_event
    0x61: cache_entry_modified_event
    0x62: cache_entry_removed_event
    0x66: counter_event
    0x80: counter_get_and_set
    0xE4: get_stream_end
    0xE6: get_stream_next
    0xE8: get_stream_start
    0xEA: put_stream_end
    0xEC: put_stream_next
    0xEE: put_stream_start

  response_status:
    0x00: success
    0x01: not_executed
    0x02: key_does_not_exist
    0x03: success_with_previous
    0x04: not_executed_with_previous
    0x81: invalid_magic_or_message_id
    0x82: unknown_command
    0x83: unknown_version
    0x84: parse_error
    0x85: server_error
    0x86: command_timed_out
    0x87: node_suspected
    0x88: illegal_lifecycle_state

  client_intelligence_type:
    0x01: basic
    0x02: topology_aware
    0x03: hash_distribution_aware

  media_type_kind:
    0x00: none
    0x01: predefined
    0x02: custom

  time_unit:
    0x00: seconds
    0x01: milliseconds
    0x02: nanoseconds
    0x03: microseconds
    0x04: minutes
    0x05: hours
    0x06: days
    0x07: default
    0x08: infinite

  hash_function_version:
    0x01: murmur_hash_2
    0x02: murmur_hash_3
    0x03: murmur_hash_3_with_segments
```

## Usage

### Generating Parsers

Install the Kaitai Struct compiler:

```bash
# macOS
brew install kaitai-struct-compiler

# Linux (snap)
snap install kaitai-struct-compiler

# Or download from https://kaitai.io/#download
```

Compile to target language:

```bash
# Java
kaitai-struct-compiler -t java hotrod.ksy

# Python
kaitai-struct-compiler -t python hotrod.ksy

# Go
kaitai-struct-compiler -t go hotrod.ksy

# C++
kaitai-struct-compiler -t cpp_stl hotrod.ksy

# Rust (community target)
kaitai-struct-compiler -t rust hotrod.ksy

# JavaScript
kaitai-struct-compiler -t javascript hotrod.ksy
```

### Using the Web IDE

1. Open [ide.kaitai.io](https://ide.kaitai.io)
2. Paste the `.ksy` schema
3. Load a Hot Rod packet capture (hex dump or `.bin` file)
4. The IDE visualizes the parsed structure overlaid on the hex dump

### Writing Test Vectors

Capture Hot Rod traffic and save as binary files:

```bash
# Capture Hot Rod traffic on port 11222
tcpdump -i lo -w hotrod_capture.pcap port 11222

# Extract TCP payload from a single packet
tshark -r hotrod_capture.pcap -T fields -e data | xxd -r -p > hotrod_request.bin
```

Then validate against the schema:

```python
from hotrod import HotrodRequest
import io

with open("hotrod_request.bin", "rb") as f:
    msg = HotrodRequest.from_io(f)
    print(f"Opcode: {msg.header.opcode}")
    print(f"Cache: {msg.header.cache_name}")
```

## Serialization Strategy

Since Kaitai Struct generates only deserializers, a complementary serialization
approach is needed. Options in order of pragmatism:

### Option 1: Schema-guided hand-written serializers (recommended short-term)

Use the `.ksy` schema as the single source of truth. Write serializers manually
in each language, but validate them against the Kaitai-generated deserializers
using round-trip tests:

```
original_bytes -> Kaitai.deserialize() -> object -> hand_written.serialize() -> round_trip_bytes
assert original_bytes == round_trip_bytes
```

### Option 2: Code generation from `.ksy` (recommended medium-term)

Write a code generator (e.g., a Python script using the Kaitai `.ksy` YAML as input)
that emits serializer code for each target language. This is a one-time
investment that pays off as the protocol evolves.

### Option 3: Symmetric IDL (future protocol version)

If a future protocol version breaks backward compatibility, consider adopting
Cap'n Proto or FlatBuffers, which provide both read and write codegen natively
and offer zero-copy performance.

## Next Steps

1. **Consolidate into a single `.ksy` file**: Combine the type fragments above
   into a single compilable `hotrod.ksy` with proper imports.
2. **Add test vectors**: Capture real Hot Rod traffic and create `.bin` test
   fixtures alongside the schema.
3. **Validate against existing clients**: Run the generated Java deserializer
   against the existing Hot Rod test suite to verify schema correctness.
4. **Generate conformance tests**: Use the schema to auto-generate protocol
   conformance test cases for non-Java clients.
5. **Integrate into CI**: Compile the `.ksy` on every protocol change to ensure
   the schema stays in sync with the implementation.
