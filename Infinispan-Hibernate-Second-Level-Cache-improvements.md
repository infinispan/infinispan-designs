# WORK IN PROGRESS...

TODO: Add about ability to embed cache configurations directly within Hibernate properties. XML snippets?

This wiki contains ideas for improvement in the Infinispan-based Hibernate Second-Level Cache implementation that address both performance improvements and lacking functionality.

# Known issues:
https://issues.jboss.org/browse/ISPN-5720 putForExternalRead may not acquire lock in invalidation mode

# Local vs Clustered

The current implementation provides a single region factory implementation, `InfinispanRegionFactory`, regardless of whether Hibernate is clustered or not. To be able to address performance improvement in different scenarios, it would help to separate the region into two regions, once for local, single node, scenarios and another for clustered multi-node environments. By doing so, each region could make the most of each use case, e.g. 

* Apply context flags relevant to the use case.
* Avoid `putForExternalRead` starting a new transaction when using local caches and `read-only` entities.

# Missing concurrent strategies

Currently Infinispan-based 2LC implementation only supports `read-only` and `transactional` concurrency strategies, and hence neither `read-strict` and `read-write-non-strict` are implemented. These two strategies are used when no JTA transaction manager is available and entities/collections need to be updated.

Implementing these concurrency strategies for local caches is relatively straightforward. For clustered environments though, you want to avoid having partially applied updates around the cluster, and using a transaction manager avoids that. Once quick way to get around this issue would be to configure the entity/collection cache with a batching transaction manager.

**UPDATE**: `read-write` strategy has been implemented. In fact the `transactional`|`read-write` switch does not matter, the appropriate way is selected according to Infinispan cache configuration. Also, when the entity is tagged as immutable, `read-only` mode is used, although it does not change much (some synchronization is required event for read-only entities to prevent staleness after removal).
**TODO**: I don't think that embedding XML into properties (as one TODO above suggests) is the best way. I would rather consider removing the XML configuration from ORM alltogether and configure the cache programmaticaly based on the strategy, and for other options (such as invalidation/replication mode) keep these in properties. Do we really need to expose locking timeouts, synchronization mode etc.?

# Consistency fixes

The problem of keeping 2LC consistent (that means, not providing stale data after DB transaction has been committed) can be summarized into 'Do not give data that are about to be updated'. This requires that each write into DB is 'wrapped' - before the DB is committed, we must invalidate the old data (so that we don't present these after DB is committed) and disable any attempts to cache anything, and after the DB is committed we have to re-enable the caching again.

The definition of stale read needs some clarification: the read can be considered stale when it happens after the DB is committed (you can find out by reading a non-cached entry from DB that was committed in the same transaction), or after the committing thread finishes the commit (that means after all synchronizations finish). When the cache is in invalidation mode (the PutFromLoadValidator-based strategies), all invalidations happen before the DB is committed, adhering to a more strict sense. In distributed/replicated modes (tombstones-based policies), we relax this a bit: while removals are executed before DB commits, updates keep the value and change it only from the synchronizations' afterCompletion. That decision can be revised yet.

These fixes incur some performance overhead compared to the previous implementation - we need two RPCs for each modification (read performance did not change). The second RPC in invalidation mode and for removals dist/repl mode is asynchronous (inserts and updates in invalidation mode are yet followed by PFER), dist/repl inserts/updates need the RPC synchronous since these are overwriting the old (stale) value.

## Invalidation-mode caches
This implementation was build on top of existing PutFromLoadValidator code, and the idea how putFromLoad operation is checked against record stored in local 'pending puts' (PP) cache persists. Since after API changes for 5.0 we can use entity IDs as keys directly (without wrapping them along with origin type information), every entity/collection cache has now its own PP cache. In the past, the entry in PP contained just the records from the thread that's about to execute putFromLoad operation, now it contains also records for each invalidator. Each record is timestamped so that any leaked record can eventually expire.

In order to implement above mentioned two phases, we have to inject custom interceptors that react on transaction being committed, for non-transactional caches we register a synchronization. This could be further optimized if there are multiple synchronizations for one transaction.

The original code was not invalidating anything when the entry was updated/inserted. As this can interact with concurrent removals, we have to invalidate the on every operation. The only exception is 'evict()' as this does not modify DB contents, therefore we can just remove the entry from cache.
As we expect that the entity will be cached after update/insert, we schedule a PFER operation if the transaction succeeds. 

Complete region invalidation is based on timestamps: during the removal of entities, nothing can be cached, and after the invalidation ends, only those reads in transactions started after the invalidation has finished (removal on all nodes has been executed) can be cached again.

## Replicated/distributed-mode caches: Tombstones
The need for keeping separate cache for pending puts has led us to an idea of using expirable entries, called tombstones for marking entity removal from the cache. Initially I thought that writing a singleton entity to the cache instead of removal would fix the problem. It would truly disable any cache updates, but there's one glitch: it would disable them until tombstone expiration, which is usually unacceptably long time.

Therefore, the tombstones are inserted before DB commit, and 'invalidated' after DB commit. If there are two concurrent removals, we have to track both of them in the tombstone and allow cache update on this entry only after all removals have finished^1^. For executing these merge (and other operations), we use `TombstoneCallInterceptor` which simulates the behaviour of PutKeyValueCommand with some added value.
We don't use PFERs for updating the cache, but put a special `TombstoneUpdate` with all the flags as PFER but without the PFER flag (since Infinispan would not read an existing entry into command's invocation context if this had the PFER flag). `TombstoneUpdate` carries the new value and timestamp of transaction start - updates with timestamps lower than the last Tombstone timestamp (the last finished removal) are discarded.

To be continued...

^1^DB should prohibit two successful removes at the same time... or not?

# Avoid Transactions

**OUTDATED**:
In the current set up, transactional Infinispan caches are needed when running Hibernate 2LC in a cluster or when using Hibernate locally with entities that need updating. Big performance improvements could be achieved if no transactions would be required to keep the cache consistent. 

For a fully non-transactional implementation to work, the following use cases would need to be addressed:

## Avoid updates/inserts overriding deletes

Without transactions, tombstones are required to know that a cached entry has been deleted. If tombstones are implemented, metadata needs to be left around so that if an update/insert comes, we can detect whether they should be applied if they're more recent, or they should not apply if they're previous to the delete itself.

**UPDATE**: Even if we know the version, it's not a guarantee that the DB transaction will succeed, so we can insert the entity only after the transaction is successful. We need^2^ to stop PFERs before the DB commit, so the version needs to be written (synchronously across cluster) before that. That leaves us this:
* before DB commit, make actual and lower versions invalid
* if TX succeeds, we may update (asynchronously) the value (if there was no other modification in the meantime)
* if TX fails, we have to revert the invalidation
If we don't update the value after tx, we've reduced the amount of RPCs needed to 1 (in successful case), but that one stored RPC will be executed by a PFER later, as we haven't cached the value in the first RPC.

^2^ We need to stop them if we want the up-to-date reads even before the modifying transaction fully finishes. If we relax this demand, we could just update the value after the transaction (with a version check to prevent overwriting more recent value), and let putFromLoads do the same. Note that current consistency tests would not detect this.

## Avoid older updates overriding newer updates

Once transactions are disabled, versioning information provided by the entity/collection itself (often dictated by the DB) can indicate which version of the entity/collection is newer than the other, and hence avoid this problem.

## Avoid partial cluster wide updates

When running in a cluster, there's a possibility that an update might not be applied in all nodes. When that happens and transactions are enabled, there's a rollback phase that makes sure no such thing happens. With transactions disabled, there's no rollback phase. If the cache is synchronous, the caller would get a failure but it'd its responsibility to either retry the update or remove the entity/collection from the cache cluster wide, which could again be partially applied. If the cache was asynchronous, the failure would go unnoticed. So, the question would be, how would a node detect that when an entity/collection is retrieved from the cache, it's stale because the database contains a newer version? The cache is there to avoid having to go the database...

**UPDATE**: Note that asynchronous caches haven't been taken into account in consistency fixes below. Any asynchronicity needs to be explicitly requested in the algorithm. If the failure happens in first phase, we rollback the whole transaction and act accordingly on the other entries. As during the update the cache is usually in invalidating state (not providing any value) a broken partial update in second phase just causes the entry to be non-cacheable until the invalidation expires.

# Improving `putFromLoad`

One of the most complex areas within the current Infinispan-based Hibernate 2LC implementation is the `putFromLoad` validator which was added to solve [HHH-4520](https://hibernate.atlassian.net/browse/HHH-4520). Effectively, it validates whether entity/collection that has been read from the database can be allowed to be put in the cache, an operation commonly known as `putFromLoad`.

To speed up `putFromLoad` operations, Infinispan uses a [`putForExternalRead`](http://infinispan.org/docs/7.2.x/user_guide/user_guide.html#_putforexternalread_operation) operation that speeds up put calls when putting data that comes from an external source. A key aspect of this operation is that it does not participate on an ongoing transaction, so there can be situations when `putForExternalRead` could store stale data in the 2LC. Example:

* Transaction 1 (T1) reads collection, results in cache miss.
* T1 goes to database to read collection
* Transaction 2 (T2) reads collection, cache miss, results in cache miss.
* T2 goes to database to read collection
* T2 calls Infinispan's `putForExternalRead` to store the collection
* T2 removes or updates collection
* T1 does Infinispan putForExternalRead to store *stale* collection

The way Infinispan 2LC implementation gets around this, while still using `putForExternalRead` is by doing the following:

1. When Hibernate reads from the cache and the entity/collection is not there, a pending put is registered in the validator entity/collection key. 
2. Once data has been read from the database, it tries to acquire a lock on that key.
3. If the lock is acquired, it's allowed to put the data into the 2LC. If the lock is not acquired, the data cannot be put in the cache.

If there's any updates to the entity/collection in the mean time, previously registered pending puts are invalidated, and as a result of that, acquiring the lock fails. These pending puts are stored in a local Infinispan cache with aggressive expiration settings, so that pending puts are removed in case there are any problems after registering them.

The above scenario can't happen when the transactions are located in different nodes when using the default Infinispan 2LC configuration which configures entity/collection caches with `invalidation`. With invalidation, `putForExternalRead` operations are not sent around the cluster since there is nothing to invalidate. The data `putForExternalRead` puts in the cache is data that comes from the database. As a result of this, there's no such risk of storing stale data into another node remotely. However, if the 2LC entity/collection cache was configured to use replication or distribution, since `putForExternalRead` replicate asynchronously in these set ups, the risk would be there.

There would not need to have this put from load validator if Infinispan's `put` was used instead of `putForExternalRead`, but for the vast majority of cases, `putForExternalRead` offers much better performance particularly for situations where data is simply loaded from the database and not updated nor removed.

Remember that the put from load validator is needed also for `read-only` entity/collections because they can be both inserted and deleted, and deletes can generate the problem explained above.

In previous performance testing, this area appeared to be a performance hotspot and it has been since tweaked to perform better. The question here is whether there's a better way to get around this stale read issue, while keeping up good performance. Tombstones and comparison based on Hibernate-provided version information would avoid a stale read overriding a delete (tombstone), or a stale read overriding an newer version (versioning).


# Remote Infinispan 2LC

So far, Hibernate has talked to Infinispan using the embedded mode, where Infinispan keeps the cached data in the same JVM where Hibernate is running. However, for a while now there has been an alternative way to interact with Infinispan caches using remote APIs that allow data to be stored in a single Infinispan Server server, or cluster of them.

There can be situations where accessing Infinispan 2LC remotely could be more suited that the current embedded way, for example:

* When data is very big, much bigger than the capacity of the JVM where Hibernate is running and data is read intensive, having a separate Infinispan Server cluster as 2LC would be very useful by lowering the amount of load on the database.
* For write heavy entities in a clustered set up, having to send SYNC invalidation messages around the cluster could be more expensive than simply talking to a single Infinispan Server instance that keeps the cache in-memory.
* When you want to run application layer where Hibernate is located in multiple nodes that are not clustered. In this case, you can't have multiple local second level cache instances running because they won't be able to invalidate each other for example when data in the second level cache is updated. In this case, having a remote second level cache could be a way out to make sure your second level cache is always in a consistent state, will all nodes in the application layer pointing to it. 
* Rather than having the second level cache in a remote server, you want to simply keep the cache in a separate VM still within the same machine. In this case you would still have the additional overhead of talking across to another JVM, but it wouldn't have the latency of across a network. The benefit of doing this is that:
** Size the cache separate from the application, since the cache and the application server have very different memory profiles. One has lots of short lived objects, and the other could have lots of long lived objects.
** To pin the cache and the application server onto different CPU cores (using numactl), and even pin them to different physically memory based on the NUMA nodes.

Until recently, implementing a remote version of Infinispan 2LC had not been considered because Infinispan's Remote Cache's were missing near caching. Without it, each Infinispan read operation would have needed to be sent to the Infinispan Server(s), adding considerable latency to even the simplest of cache checks. However, Infinispan's Remote Cache recently had Near Caching added so that clients can keep small local caches client-side, with agressive eviction policies, that speed up remote data access while avoiding stale data by reacting to events coming from the Infinispan server. This significant improvement makes a remote Infinispan 2LC implementation a viable solution.

**NOTE**: The near cache is designed to be updated asynchronously after the update happens on server. That makes the options somewhat limited - we cannot guarantee non-stale reads. Therefore, I think that when using near cache we can support only nonstrict-read-write mode (which does not give any guarantees).