# Transactions
## Transactions

### How Hot Rod does it

Hot Rod transactions use a **client-side buffering** model with XA-style 2-phase commit:

1. The client starts a transaction locally and generates an XID (format `0x48525458` / "HRTX").
2. All operations (put, get, remove) execute against a local `TransactionContext` — **nothing
   goes to the server** during the transaction. Each entry tracks a control byte indicating
   whether the key was read (`READ`), never read (`NOT_READ`), or didn't exist
   (`NON_EXISTING`), plus the version at read time.
3. At commit time the client sends a single **PREPARE_TX** request containing ALL modifications:
   ```
   XID, onePhaseCommit flag, recoverable flag, timeout,
   modification_count,
   for each modification:
     key, control_byte, [version_read], [lifespan/maxIdle], [value]
   ```
4. The server's `PrepareCoordinator` starts an `EmbeddedTransaction`, validates each
   modification (version check for optimistic, write-lock acquisition for pessimistic),
   applies all writes via the transactional cache, and runs `tx.runPrepare()`.
5. If successful, the server records the state as `PREPARED` in the replicated
   `GlobalTxTable`. The client then sends **COMMIT_TX** (or **ROLLBACK_TX**).
6. Commit/rollback is coordinated via the `GlobalTxTable` — if the originator is still alive,
   the command is forwarded to it; if it left the cluster, the command is broadcast.

**Key characteristics:**
- Reads within the transaction return the locally-buffered value (read-your-writes).
- The server only sees the transaction at prepare time — no server-side state during the
  transaction's active phase.
- Optimistic validation: server checks that `version_read` matches the current version.
- Cross-cache transactions: a single XID spans multiple caches; the `GlobalTxTable` tracks
  per-cache state via `CacheXid` keys.
- Recovery: timed-out `PREPARED` transactions can be recovered via `FETCH_TX_RECOVERY`.

### Why client-side buffering doesn't fit gRPC's model

The user's requirement is a **begin/commands/commit** conversational model where commands are
issued individually and executed on the server. Hot Rod's client-side buffering model has
properties that conflict with this:

- Reads must be sent to the server to get current values, but writes are invisible to the
  server until prepare time — the client must maintain a write-set overlay.
- The client must track versions for every key read, adding client-side complexity.
- The entire write-set is sent in one large prepare message — this is a batch, not a
  conversation.
- Client libraries in every language must implement the full `TransactionContext` logic
  (buffering, version tracking, conflict detection), significantly raising the bar for
  client implementations.

### Proposed design: bidirectional streaming transaction

A transaction is modelled as a **single bidirectional streaming RPC**. The client streams
operations and the server streams results, with the stream's lifetime bounding the
transaction:

```protobuf
service Cache {
  // Execute a sequence of operations within a transaction.
  // The stream begins with a Begin message and ends with Commit or Rollback.
  // If the stream is interrupted (client disconnect, error), the server
  // rolls back automatically.
  rpc ExecuteTransaction(stream TransactionOp) returns (stream TransactionResult);
}

message TransactionOp {
  oneof op {
    TxBegin begin = 1;
    TxPut put = 2;
    TxGet get = 3;
    TxRemove remove = 4;
    TxCommit commit = 5;
    TxRollback rollback = 6;
  }
}

message TransactionResult {
  oneof result {
    TxBeginResult begin_result = 1;
    TxPutResult put_result = 2;
    TxGetResult get_result = 3;
    TxRemoveResult remove_result = 4;
    TxCommitResult commit_result = 5;
    TxRollbackResult rollback_result = 6;
  }
}

message TxBegin {
  string cache_name = 1;
  int64 timeout_ms = 2;       // transaction timeout
}

message TxBeginResult {
  string transaction_id = 1;  // server-generated, for diagnostics/logging
}

message TxPut {
  bytes key = 1;
  bytes value = 2;
  int64 lifespan_ms = 3;      // 0 = default
  int64 max_idle_ms = 4;      // 0 = default
}

message TxPutResult {
  bytes previous_value = 1;   // previous value, if any
}

message TxGet {
  bytes key = 1;
}

message TxGetResult {
  bytes value = 1;            // null if not found
}

message TxRemove {
  bytes key = 1;
}

message TxRemoveResult {
  bool removed = 1;
  bytes previous_value = 2;
}

message TxCommit {}

message TxCommitResult {
  bool success = 1;
  string message = 2;        // error detail if !success
}

message TxRollback {}

message TxRollbackResult {
  bool success = 1;
}
```

### Server-side execution model

The server uses Infinispan's **embedded transaction manager** directly. Each streaming RPC
maps to an embedded transaction:

```
Stream opened
  ↓
TxBegin received:
  EmbeddedTransactionManager.begin()
  → creates EmbeddedTransaction, associates with current thread
  ↓
TxPut received:
  cache.put(key, value, metadata)
  → executes within the transaction context
  → Infinispan handles distribution internally (forwards to key owners)
  → write locks acquired (pessimistic) or deferred (optimistic)
  ← returns previous value
  ↓
TxGet received:
  cache.get(key)
  → reads within the transaction context
  → sees uncommitted writes from this transaction (read-your-writes)
  ← returns value
  ↓
TxCommit received:
  EmbeddedTransactionManager.commit()
  → triggers Infinispan's internal 2PC across involved nodes
  ← returns success/failure
  ↓
Stream closed
```

**Key advantage**: the server-side model reuses Infinispan's entire embedded transaction
infrastructure (locking, 2PC, write-skew detection, deadlock detection, recovery) without
needing the Hot Rod-specific `PrepareCoordinator` / `GlobalTxTable` machinery. The gRPC
server simply operates as an embedded client would.

### Why bidirectional streaming

The alternative — separate `Begin`, `Commit`, `Rollback` RPCs with a transaction ID in
metadata on regular cache RPCs — has fundamental problems:

1. **Transaction affinity**: an embedded transaction is bound to the `TransactionManager`
   instance on a specific server. If the client routes a `Put` within a transaction to a
   different server (e.g. because key-based routing sends it to the key's primary owner),
   that server has no knowledge of the transaction. The receiving server would need to
   forward the operation back to the transaction coordinator, adding complexity.

2. **Thread association**: Infinispan's `EmbeddedTransactionManager` associates transactions
   with threads. Separate unary RPCs may execute on different threads, requiring manual
   transaction context propagation.

3. **Lifecycle management**: if the client crashes between `Begin` and `Commit`, there's no
   natural cleanup trigger. With a stream, the server detects the stream close and rolls
   back automatically.

The bidirectional streaming model solves all three: all operations flow through a single
stream to a single server (the coordinator), can be pinned to a transaction context, and
the stream's lifecycle provides natural cleanup.

### Routing during transactions

Within a transaction stream, **all operations go to the coordinator server** — the server
where the stream is established. This server uses the embedded cache API, which handles
distribution internally:

- For a `put(k, v)`: if key `k` is owned by another node, the embedded cache forwards the
  write internally via Infinispan's clustering layer.
- For a `get(k)`: reads may be forwarded to the primary owner, or served locally if the
  cache mode permits.

This means **the client does not use key-based routing within a transaction**. The client
should send the `ExecuteTransaction` stream to any cluster member (or its preferred
coordinator). The internal forwarding cost is acceptable because:

- Transactions are relatively infrequent compared to non-transactional operations.
- The embedded cache's internal forwarding is highly optimized (same code path as embedded
  applications).
- Avoiding client-side routing within transactions dramatically simplifies client
  implementations.

### Cross-cache transactions

A single transaction stream can span multiple caches by interleaving operations that
specify different cache names:

```protobuf
message TxPut {
  string cache_name = 1;     // optional — defaults to the cache from TxBegin
  bytes key = 2;
  bytes value = 3;
  int64 lifespan_ms = 4;
  int64 max_idle_ms = 5;
}
```

The embedded `TransactionManager` naturally enlists multiple caches in the same transaction.
At commit time, Infinispan's internal 2PC coordinates across all involved caches and nodes.

### Error handling and automatic rollback

| Scenario                      | Behaviour                                        |
|-------------------------------|--------------------------------------------------|
| Client sends `TxRollback`     | Server rolls back, sends `TxRollbackResult`      |
| Client disconnects mid-tx     | Server detects stream close, rolls back           |
| Operation fails (lock timeout)| Server sends error in `TransactionResult`, tx     |
|                               | marked rollback-only; client must send `TxRollback`|
| Commit fails (write-skew)     | `TxCommitResult.success = false` with detail      |
| Transaction timeout           | Server rolls back, sends error, closes stream     |

The bidirectional stream gives the server a reliable cleanup signal: if the stream
terminates for any reason without a `TxCommit`, the transaction is rolled back.

### Comparison with Hot Rod

| Aspect                    | Hot Rod                           | gRPC (proposed)                    |
|---------------------------|-----------------------------------|------------------------------------|
| **Buffering**             | Client-side                       | Server-side (embedded tx)          |
| **Server state during tx**| None until prepare                | Active embedded transaction        |
| **Read-your-writes**      | Client-side overlay               | Native (embedded cache)            |
| **Wire protocol**         | Single large PREPARE message      | Streaming ops, one at a time       |
| **Routing during tx**     | N/A (all sent at prepare)         | All to coordinator, internal fwd   |
| **Commit**                | Explicit PREPARE + COMMIT RPCs    | Single `TxCommit` on stream        |
| **Cleanup on disconnect** | Timeout-based reaper (60s)        | Immediate (stream close)           |
| **Client complexity**     | High (version tracking, buffering)| Low (just stream ops)              |
| **Recovery (XA)**         | Full XA recovery protocol         | Not needed (server-managed)        |
| **Lock holding duration** | Short (only during prepare)       | Full transaction duration           |

### Trade-offs and considerations

**Lock holding duration**: the primary trade-off of the server-side model is that write locks
are held for the entire transaction duration (from the first write until commit/rollback),
rather than only during the prepare phase as in Hot Rod. Long-running transactions will block
other operations on the same keys. Mitigations:

- **Transaction timeout**: enforced server-side via `TxBegin.timeout_ms`. Default should be
  short (e.g. 30 seconds).
- **Client best practice**: transactions should be short-lived. Batch all operations and
  commit quickly.
- **Monitoring**: expose transaction duration metrics via the existing metrics framework.

**Scalability**: each active transaction holds an embedded transaction context and
potentially write locks. Under high concurrency, this uses more server resources than Hot
Rod's client-side buffering model. However, this is the same cost as any embedded application
using transactions — the infrastructure is designed for it.

**Simplicity for client implementations**: this is the major advantage. A gRPC client in any
language (Go, Python, Rust, C++, etc.) needs only to open a bidirectional stream and send
protobuf messages. No version tracking, no conflict detection, no write-set management.
This dramatically lowers the bar for client library implementations.

