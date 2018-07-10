# Second level cache design

Infinispan implementation of second level cache (2LC) supports all Hibernate [access types](https://github.com/hibernate/hibernate-orm/blob/master/hibernate-core/src/main/java/org/hibernate/cache/spi/access/AccessType.java) but supported combinations of access type and cache modes vary. 2LC can work in invalidation mode, replicated/distributed mode or locally - this design document will focus on clustered caches since local caching is usually implemented as invalidation mode just without sending any messages to other nodes.

In the past 2LC used Infinispan transactional modes to prevent publishing changes uncommitted in the JTA transaction. During Hibernate ORM 5.0 development the caching was reimplemented, integrating directly into Hibernate's unified JTA/JDBC SPI. There are two motivations: optimized performance (non-transactional cache is generally faster) and ability to run with JDBC-only transactions. The support for transactional mode in Infinispan was kept until (not including) Hibernate ORM 5.3: we've dropped the burden of maintenance at that point.

There are two key features that should be held in mind during any changes:

1) Reads must never block. Infinispan naturally supports non-blocking reads (*1), but when there's a cache miss Hibernate loads the entry from database and asks 2LC to store it - we call this write a `putFromLoad`. If it's not possible to update the cache immediatelly (usually because the entry is locked locally) the thread must not be blocked; we'll rather discard the update. Some cache modes require replication to another node; that replication must happen asynchronously.

2) `TRANSACTIONAL`, `READ_WRITE` and `READ_ONLY` access type prevent the cache from returning a value that wouldn't be read from the database. The JTA/JDBC transaction must keep its ACID semantics which we interpret as *serializable isolation level*. As there might be mixed access within an transaction (A is read from DB and B from cache) we must prevent inconsistencies even in the short window between updating DB and the cache, usually by invalidating the entry in cache before the DB commit **and** preventing any further updates until the transaction commits/rolls back.
This property should be preserved even in the presence of an error (e.g. RPC failure). It's ok to roll back the transaction when we are not able to invalidate the entry. After the DB commit the error handling happens on a best effort basis - we might keep the entry in a state when it's not ready to accept new updates (locked) even after the transaction is completed, though we should unlock it eventually (usually based on a wall-clock timestamp).

`NONSTRICT_READ_WRITE` access type must not disclose uncommitted data and prevents reading stale data from the thread that executed the update. However it does not lock the entry and therefore it does not guarantee serializability as described above. Also since the cache is updated only after DB commit, in the presence of failures it's possible that the cache won't be updated at all - we don't have the opportunity to roll back the transaction and therefore we might encounter stale read even in the same thread.

In addition to these two, all external calls should result in at most one in-place update. Infinispan now offers API powerful enough to execute this, so we should avoid any putIfAbsent - replace loops that involve RPC.

## 2LC metadata storage

Infinispan stores keys, values and metadata. In 2LC case the keys are either entity ids (when `hibernate.cache.keys_factory=simple` - this is not default, though) or tuples that carry the entity type, tenant id and the entity id (`hibernate.cache.keys_factory=default`). Metadata is not used extensively - 2LC uses only expiration and maxIdle. It gets a bit more complicated with the values.

Traditionally 2LC implemented only the invalidation mode and data were expected to be evicted from the cache as needed. The entity cache contained only Hibernate-dehydrated entities as values (*2), the 2LC metadata (such as the fact that entry is locked and cannot be updated, or that we had a cache miss and expect a `putFromLoad` - so called *pending put*) was stored in another local-mode cache - for `org.my.Entity` that would be called `org.my.Entity-pending-puts`. This cache was accessed solely through `PutFromLoadValidator`. The reason to keep this as another cache rather than plain concurrent hash map is that we use expiration on this map to avoid running out of memory when something is not cleaned up properly (the data in this cache is usually short-lived).

Replicated/distributed strategies went in a different direction, keeping both in single cache. Since it is not safe to deliberately evict locking information, this cache must not be configured for eviction (*3). Modifying the values in-place calls for functional commands but these were not available when the code was written, therefore regular `PutKeyValueCommand`s were used and `CallInterceptor` was replaced by an interceptor that included special handling for applying the commands. Luckily, functional commands are now part of Infinispan and recent versions use this feature to apply the modifications.

## Cache modes

While 2LC heavily modifies how Infinispan works, the basic idea behind cache modes stays: invalidation mode stores data only locally and invalidates other modes, replicated mode keeps all data on all nodes (therefore even `putFromLoads` push data to other nodes) and distributed mode works as replicated mode but updates only a subset of the nodes based on the key - the owners.

What is different is the order of replication. Vanilla Infinispan needs to use single order of updates on all owners and does so (in non-tx mode) by routing the update to primary owner and replicating to the other owners from there. In case of 2LC and we gain performance by omitting this step and applying the update locally (*4) and then replicating to all owners immediatelly. Each strategy ensures this in a different way:

### Invalidation mode

In invalidation mode the value is never replicated to other nodes, so we don't need to deal with any ordering - before DB commit we synchronously broadcast the `BeginInvalidationCommand` to all other nodes and after the transaction we'll asynchronously broadcast the `EndInvalidationCommand`. Different nodes can't end up with different values because after receiving the begin-invalidation the node won't update the cache in the second phase, it will only unlock the entry.

### Replicated/distributed mode with `TRANSACTIONAL`, `READ_WRITE` or `READ_ONLY` access:

The regular ordering is not necessary because we have only few types of updates:

* lock & invalidate (`Tombstone` class)
* unlock & update value (`FutureUpdate`)
* putFromLoad (`TombstoneUpdate`)

If two condurrent lock & invalidate are issued from two different nodes at the same time, the result is always the same: the entry gets invalidated and locked. Both attempts to lock the entry are recorded and the entry becomes available for caching only after both transactions are completed.
// TODO: Reading the code now it seems we may have a flaw here - since the unlock & update is replicated asynchronously it might come long after subsequent update, and update the cache with a stale value. It's possible that this did not manifest before because a database using blocking locks would delay the second update enough, or roll it back completely when it's using optimistic locking scheme.
// The difference to invalidation mode above is that the `FutureUpdate` is scheduled in a synchronization and does not get invalidated by another lock & invalidate. We should probably drop value from the `FutureUpdate` when the entry is locked even after it gets applied.

The putFromLoads are safe to be applied because when the entry is locked or a value is present it is simply discarded. When the entry is removed we'll keep a `Tombstone` with removal timestamp and apply the `TombstoneUpdate` only if its timestamp (time when the originating transaction started) is higher than the removal timestamp.

Note that Infinispan (as of now) does not guarantee that each command will be invoked on each node only once. That's why the commands carry random 128 bit identifiers (UUIDs) and store these in `Tombstones` rather than keeping just a counter.

### Replicated/distributed mode with `NONSTRICT_READ_WRITE` access:

Using this access type is allowed only on versioned entities. Since database already gives us the order in which these should be applied we can rely on that, and apply the update only if the current version (be it from putFromLoad or modification) is higher. For removals we need to keep tombstones (`VersionedEntry` in this case), the version comparison considers a deleted entry of version X higher than version X itself but lower than any other version Y where X < Y.

## Locking

In the old days of transactional invalidation mode the locking was happening through regular locking commands on primary owners. This does not play well with the `putFromExternalRead` command which is used for the `putFromLoad` and is always executed only locally - that means that it does not acquire lock if the current node is not the primary owner.

// TODO: it seems that this is still broken with some exception of putFromLoad/beginInvalidation interactions piggy-backing on the locking in `PutFromLoadValidator`. ISPN-9075 introduces `LockingInterceptor` (see below) to non-transactional invalidation caches, too - that should sort it out.

In replicated/distributed mode we also acquire lock only on primary owner. Older Infinispan versions used to hold that lock until all other owners were updated, newer ones just make sure to update other owners in the same order and unlocks immediatelly.

Since we don't use these means to isolate updates, we need to lock the entry on all owners. However it is not possible to keep the lock locked during a synchronous replication - besides limiting throughput it could cause deadlocks when two nodes lock the entry concurrently and then wait for synchronous update on the other node.
This problem is tied to the architecture of the interceptor stack, where we lock higher in the stack, apply the command at the bottom and the replication happens unroll the stack on the way up to locking interceptor. The way out is returning a `CompletableFuture` representing the replication in the `UnorderedDistributionInterceptor` and passing that up the stack. `LockingInterceptor` unlocks the entry and checks if the command return value is a `CompletableFuture` - in this case it does not continue the control flow until the replication is completed. That way we keep the synchronous behaviour of the command even though we need to block at different phase of the interceptor stack.

With some optimizations in ISPN-9075 it gets even more complex. Certain operations do not require us to wait for the replication to other nodes but we want to wait until the change is applied locally - an example is the `Tombstone` before the DB commit in replicated mode. While the invalidation must be replicated synchronously (we must ensure that all other nodes have the entry invalidated before committing DB) we need to ensure it was replicated only before the transaction starts committing - other entries can be invalidated in parallel to that.
However we want to make sure that the update is applied locally (otherwise we could read stale data from the cache) before the user update is completed. We can't tell if the local update was already applied when the `writeMap.eval(...)` call returns the `CompletableFuture` (which will be completed only after the sync replication), but we can expect that it will be applied most of the time. Therefore we use the `Tombstone` itself as a marker - `CompletableFunction`. Before delaying the control flow after unlock the `LockingInterceptor` marks this `ComplatableFunction` as applied and we know that it's safe to proceed even when the CF is not done yet. If the `CompletableFunction` is not marked we have no means to selectively wait for the local changes to be applied and we will wait until the CF gets completed, including the replication.

We use similar technique in invalidation mode, too, but there we don't use functional commands with custom function that can implement the `CompletableFunction`. Therefore we abuse the `FORCE_WRITE_LOCK` flag - removals before DB commit are invoked with this flag, but `LockingInterceptor` ignores its original meaning and instead removes the flag as a marker for local completion.

Note that you could find other uses of this flag to differentiate transactional removal and non-transactional evict. The former uses a begin-invalidation/end-invalidation pair while the later simply wipes out the entry from cache (on both this and remote nodes) and invalidates all current ongoing operations with this entry.

## Query caching and timestamps

Queries required transactional caches in the past, too. To not expose the uncommitted query result we register a synchronization that writes the query result into cache after transaction commit. The result is also cached locally (because users often expect second query in the same session to be satisfied from 2LC) in a Map<Session, Map<Query SQL, Query result>>.

ClusteredTimestampsRegionImpl does not retrieve timestamps from the cache directly, instead it registers listener for modification on the cache and inserts the values to a concurrent hash map which is used for reading later on. This probably is an old performance optimization for JBoss Cache that may have lost the reason, but the truth is already hidden beneath the sands of time.

---

(1) even at the risk of providing inconsistent image: when ran in transactional mode and a transaction T1 updates two keys A -> 1, B->1 to A -> 2, B -> 2, second transaction T2 may read A -> 2, B -> 1 or vice versa. This breaks the traditional ACID isolation which protects reads with short locks and writes with long locks.

(2) collections and native ids are stored in the same way

(3) this is a limitation that might be removed in the future by marking the non-evictable items appropriatelly

(4) in distributed mode only if the node is an owner
