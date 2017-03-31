This design doc is out of date - please refer to [package documentation](https://github.com/rvansa/infinispan/blob/ISPN-6645/core/src/main/java/org/infinispan/scattered/package-info.java)
 that's part of [preview PR #4458](https://github.com/infinispan/infinispan/pull/4458).

## Idea
Distributed caches have fixed owners for each key. Operations where originator is one of the owners require less messages (and possibly less roundtrips), so let's make the originator always the owner. In case of node crash, we can retrieve the data by inspecting all nodes.

To find the last-written entry in case of crashed primary owner, entries will keep write timestamps (counter value rather than wall-clock time) in metadata. These timestamps also determine order of writes; we don't have to use locking anymore.

For the time being, let's implement only the case resilient against one node failure (equivalent to 2 owners in distributed cache).

* + faster writes
* + no locking contention
* - reads always have to go to the primary (slower writes in small clusters)
* - complex reconciliation during rebalance

## Name
I believe that this behaviour definitely desires its own cache mode, not just a configuration attribute. As much as I'd like to call it Radim's Cache as Bela proposed, for practical reasons let's use Scattered Cache. If anyone has better name, please suggest it.

## Operations
I'll assume non-transactional mode below. Transactional mode is quite similar, though (but non-tx will be POCed/implemented first).

We need to keep tombstones with timestamp after entry removal. Luckily, these tombstones have limited lifespan - we keep them around only until the invalidations are applied on all nodes.

The timestamps have to grow monotonically; therefore the timestamp counter won't be per-entry but per segment (as tombstone will be eventually removed, per-entry timestamp would be lost).

### Single entry write (put, getAndPut, putIfAbsent, replace...)
* originator == primary owner:

1. Primary increments timestamp for segment
2. Primary commits entry + timestamp
3. Primary picks one node (let's say the next member) and sends backup RPC
4. Backup commits entry
5. Backup sends RPC response
6. Primary returns after it gets the response
7. Primary schedules invalidation of entry with lower timestamps
 * This requires just one sync RPC
 * Selection of backup could be random, but having it ~fixed probably reduces overall memory consumption
 * Updating value on primary before backup finishes does not change data consistency - if backup RPC fails, we can't know whether backup has committed the entry and so it can be published anyway.

* originator != primary owner

1. Origin sends sync RPC to primary owner
2. Primary checks op if conditional
3. Primary increments timestamp for segment
4. Primary commits entry + timestamp
5. Primary returns response with timestamp (+ return value if appropriate)
6. Origin commits entry + timestamp
7. Origin schedules invalidation of entry with lower timestamps
 * Invalidation must be scheduled by origin, because primary does not know if backup committed

### Single entry read
* originator == primary owner: Just local read
* originator != primary owner:

1. Origin locally loads entry + timestamp with SKIP_CACHE_LOAD
2. Origin sends sync RPC including the timestamp to primary
3. Primary compares timestamp with it's own
 * If timestamp matches, value is not sent back
 * If timestamp does not match, value + timestamp has to be sent 

* Optional configuration options:
 * Return value after 1) if present (risk of stale reads)
 * Store read value locally with expiration (L1 enabled) - as invalidations are broadcast anyway, there's not much overhead with that. This will still require RPC on read (unless stale reads are allowed) but not marshalling the value.

### Multiple entries writes

Entries for which this node is the primary owner won't be backed up just to the next member, but to a node that is primary owner of another entries in the multiwrite. That way we can spare some messages by merging the primary -> backup and origin -> backup requests.

### Multiple entries reads

Same as single entry reads, just merge RPCs for the same primary owners.

## Invalidations

It would be inefficient to send invalidations (key + timestamp) one-by-one, so these will be merged and sent in batches. Invalidations from other nodes can remove scheduled 'outdated' invalidations, but it requires additional complexity and the overall gain is debatable.

## State Transfer

### Node leave/crash

Operations after node crash are always driven by the new primary owner of given segment

* If it was the primary owner in previous topology:
 * Replicate all data from this segment to the next node 
 * Send invalidate request with all keys + timestamps to all nodes but the next one
 * The above is parallel and processed in chunks.
 * Write to entry can proceed in parallel with this process; ST-invalidation cannot overwrite newer entry, though write-invalidation can arrive before the ST-replication (then we would have 3 copies of each entry).
  * After chunk is replicated (synchronously), primary owner compares actual timestamps with those replicated and sends invalidations to the backup with the modified timestamps
* If the primary owner has just become a new primary owner:
 * Synchronously retrieve highest timestamp for this segment from all nodes
  * All writes are blocked until the timestamp is retrieved
 * Request all nodes to send you all keys + timestamps from this segment (and do that locally as well)
 * Compute max timestamps for each key
 * Retrieve values from nodes with highest timestamp, send invalidation to others
 * If there is a concurrent write during the retrieval, primary owner has to retrieve key + timestamp (+ value if needed) from all nodes, write can proceed only after all nodes respond
  * Second write during the ST to the same entry can be processed without the all-nodes-read. This can be achieved by storing the timestamp retrieved at the start of ST - if local entry timestamp is higher than this value, the write can safely proceed.

### Clean rebalance (after join, no data is lost)

If node has become primary owner:
* Retrieve segment timestamp counter value + old owner starts rejecting further reads & writes
* Send request for all keys + timestamp + value to primary owner
* Update local values as these come
* Write is similar to the crash case, but the value is retrieved just from primary owner
* Request old owner to remove/L1-ize all entries

### Crash during clean rebalance

This shouldn't be different from regular crash, but there might be technical difficulties to cancel the ongoing ST. Node is always considered a new primary owner for a segment that had unfinished ST, since the previous new owner did not have all entries but had some new modifications.

## Open questions

* Do we need remote-thread pool when we don't use locks?

