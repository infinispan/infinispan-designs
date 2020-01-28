Conflict Resolution Performance Improvements
====================
Currently conflict resolution (CR) performance is dog slow, due to the following:

1. All segments and their cache entities are checked
    - Performance degrades rapidly as the size of the cluster increases

2. Centralised CR
    - The merge coordinator is responsible for comparing all cache entities
        - In a distributed cache, this requires the coordinator to retrieve all segments from both partitions in order
        to compare them
    - User triggered CR occurs on the node the request is made

3. No parallelism
    - A single segment is processed at a given time in order to avoid the centralised CR coordinator suffering a OOM exception

# Proposals
The following proposals are listed in the order of the anticipated performance benefit, with those with greatest impact
presented first.

## Maintaining a Merkle tree
To minimise the number of entries that need to be checked during CR, we should maintain a Merkle tree of cache entry hashes
on a per segment basis, with the root of the tree being a hash of all the entry hashes in the segment. Given the root hash
of a segment from three different nodes, we can determine that no conflict exists simply by checking if all three values
are equal. If any of the three hashes differ, then it's then necessary to compare the segment's entries and perform CR.
Amazon Dynamo DB utilises this technique https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf.

The simplest solution for producing such a tree, is to simple have a tree of depth 2. Where each cache entry is a leaf node
and the root is a hash of all leaf hash values. This can be easily implemented by simply iterating over the segment container
in order to calculate the hash of individual entries and the combined hash.

TRACKED BY: [ISPN-8412] https://issues.redhat.com/browse/ISPN-8412

### Hash to use
MurmurHash3?

### Calculating the entry hash
Simply hash the `.hashcode()` returned by the `InternalCacheEntry` implementation.

The `equals` and `hashcode` methods of our `InternalCacheEntry` implementations should be updated to take into account
the `EntryVersion` stored in the Metadata. Currently this is ignored by CR, however including the check would allow the
following conflicts to be resolved deterministically via `LATEST_VERSION` resolution strategy:

```java
put(k, v, 1)  // 1. put k/v with version 1
put(k, v', 2) // 2.
put(k, v, 3)  // 3. Missed by partition 1
```
By comparing the entry versions during CR, it's possible to ascertain that the original value missed by a partition in
step 3 is in fact the latest version.

Even if a different merge strategy was used, maintaining an entries `EntryVersion` value is necessary in order for 
HotRod conditional operations to work as expected for the winning value post CR.

### Calculating the tree
The computation of non-leaf nodes should only occur at the start of CR, as the additional iteration of entries would
adversely affect cache write operation performance. The cost of calculating invidual leaf node hashes should be minimal,
depending on the hash used, so this could be computed actively or lazily.

Cache operations that occur during the CR phase should be treated as the latest value and should overwrite any writes
that occurr as part of CR. Therefore, once the tree has being created at the start of CR it's state should be immutable
until CR is resolved/aborted.

### Increasing Merkle Tree Depth
A segment root with all entries being it's children is not very efficient when a segment contains a large number of values,
as when an inconsistency is discover, it's still necessary to send all entries within the segment over the wire. Increasing
the depth of the tree means that it's possible to perform more fine-grained consistency checks, with a larger tree depth
and smaller amount of leaf nodes resulting in less entries having to be sent over the wire when inconsistencies do occur.

#### Constant lookup/removal tree - Depth 3
This is a simple implemenetation which adds an additional layer of depth to the tree in order to increase CR granularity:

* A segment is further divided into `X` nodes, which are the root's childen
  - These nodes represent a range of a cache's `key` hashcodes, e.g. 0..5, 6..10 etc
  - `(hash(key) & Integer.MAX_VALUE) \ X`
  - MurmurHash3 can also be used here
  - Where `X` could be configurable, but should probably just be an internal constant

* Each node contains:
    - A Map to store leaf nodes, which enables amortised constant lookup and to accomodate `.hashcode()` conflicts on keys
    - A node hash field containing a hash of all the leaf node's hashes combined

* The tree, excluding leafs, can be represented as a int and int[X] (Root+X hashes) in a RPC

* During CR if two tree's root values don't match, it's then possible to compare all `X` hashes and only request the indexes
of the array which have conflicting hashes. At which point the participating nodes only return the InternalCacheEntries
associated with those nodes.

* Once the hash tree has been created for CR, it's possible to cache the hashes until a subsequent write operation. Upon
a write operation it's necessary to invalidate the root hash and the hash of the child node containing the entry. During
the next round of CR it's then only necessary to recalculate the invalidated nodes and the root.

* Logic for processing write operations can be implemented as an interceptor which interacts with the conflict manager.

### Offline Backup Consistency Check
The Merkle tree hashes can also be utilised to ensure the consistency of data being dumped to an offline backup. When a
user initiates a backup:

1. The cluster initiates CR to ensure that no inconsistencies exist before the backup
2. A new Merkle tree is created, with the root node being the hash of all primary replica segment hashes
3. The root hash is stored as part of the backup metadata
4. When a cluster is restored from a backup, the clusterwide Merkle tree can be recreated and the new root hash can be
compared to the value stored in the metadata.

> Utilising Merkle trees in this manner would mean that it's not possible to make changes to how entries are hashed
and how the tree is created without an appropriate migration strategy between Infinispan versions.

## Distributed CR processing
Instead of CR being coordinated by a single node, the coordinator should instruct the primary owner of each segment to
initiate CR. As it's likely that a single node will be the primary for multiple segments in a small cluster, we should
still limit each node to executing CR for a single segment at a time.

Distributing CR also benefits from the use of Merkle trees, as it means that the primary owner in the preferred partition
who is coordinating the CR would not have to send their tree over the wire.

TRACKED BY: [ISPN-9084] https://issues.redhat.com/browse/ISPN-9084

## Prioritise segments based upon requests during merge
During a partition merge, if CR is in progress and a request is made on a specific key before it's segment has been processed,
then we should perform CR on the key in place.

The advantage is that conflicts on an active entitiy are resolved sooner, furthemore, if (##<<Maintaining a Merkle tree>>) is
implemented, it potentially could result in a conflict(s) being resolved before the segment is checked, meaning all entries
in the segment do not have to be retried.

The disadvantages are the increased complexity, which will only increase if (#Distributed CR processing) is implemented.

TRACKED BY: [ISPN-9079] https://issues.redhat.com/browse/ISPN-9079
