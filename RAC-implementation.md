Infinispan Reliable Asynchronous Clustering
===========================================

Original Document: [RAC: Reliable Asynchronous Clustering](RAC%3A-Reliable-Asynchronous-Clustering.asciidoc)
Implementation: Pedro Ruivo, 2019

# Changes from the original design

* The `Updaters` are the owners of the `key` and the `primary-owner` is responsible to send 
the updates to the remote site.

* The *Conflict Resolution* was re-done, and it uses versioning (`vector-clock`) to detect conflicts and duplicates.
It will be described later in this document.

# Overview

This section describes a short overview of the algorithm.

When a `key` is updated (doesn't matter if it is a transaction or not), all the owners keeps track the `key` by 
storing it in a queue.
Periodically, the `primary-owner` iterates over all `keys` queued and sends its value to the remote site.
To fetch the values, it peeks the `DataContainer` and `CacheWriter`.
If the value isn't found, it sends a `remove` request to the remote site, 
otherwise an update request with the `key` and its value.

After a successful replication (and `ack` is sent back from the remote site), the `primary-owner` sends a `cleanup`
request to the `backup-owners`.
They drop the `key` from the queue.

Note that, the `lock` for the `key` isn't acquired during this process.

# Conflict Resolution

The conflict happens when two or more sites update the same `key` concurrently. 
It uses `vector-clocks` to detect such scenarios and perform the conflict resolution to decide which
update stays and which update is discarded.

The vector clock has the following structure:

```
{
  site-1: [topology, version]
  site-2: [topology, version]
  ...
}
```

The `site-1` and `site-2` are the site's names as configured in `RELAY2`.

The `topology` is a number associated to the topology changes for that site. 
When a node joins/leaves/crashes, the `topology` increases and the `version` resets to `0`.

The `version` is a number that increments on each update for a segment. 
To improve the concurrency and reduce the memory footprint, the `version` is associated to a segment,
instead of having one per `key`.

*Note:* The `clear` operation is destructive, and it doesn't generate any new `version` neither checks for conflicts!

### Version comparator

To define the order for a `[topology, version]` pair, it compares the elements. 
The `topology` is compared first and, if `p1.topology` < `p2.topology`, than pair `p1` is less than pair `p2`.
If the `topology` is the same, then `version` with the same logic as above to determine the ordering.
Examples:

```
[1,10] is less than [2,0]
[1,11] is higher than [1,10]
```

Finally, to compare `vector-clocks`, it compares the pairs for each site.
If for all sites, the pair is lower, then the `vector-clock` is less than the other. 
On other hand, if for some sites the pairs are less  and for other the pair is higher, then we have a conflict.
Example:

.vector-clock 1
```
{
  "LON": [1,1]
  "NYC": [0,0]
}
```  

.vector-clock 2
```
{
  "LON": [0,0]
  "NYC": [1,1]
}
```


### Resolving conflicts

The current conflict resolution algorithm uses the site names to resolve the conflict.
It uses the lexicographic order to determine which update wins.
In the example above, the update from site `LON` will be applied, and the update from site `NYC` will be discarded.

The future plan includes a SPI so users can do their own conflict resolution.
See [ISPN-11802](https://issues.redhat.com/browse/ISPN-11802).

### Tombstones

When a `key` is removed, it stores a `tombstone` with the `vector-clock` corresponding to that update operation.
The `primary-owner` sends the `tombstone` to the remote site, together with the `remove` request,
 in order the remote site to perform a correct conflict resolution. 

# Implementation details

## Classes

### `IracManager`

It is responsible to keep track the `keys` updated and to send them to the `remote site`.

### `IracVersionGenerator`

It is responsible to generate the `vector-clock` and to store the `tombstones`.

### `*BackupInterceptor`

It intercepts the `WriteCommand` and notified the `IracManager` with the `keys` modified by them. 

### `*IracLocalInterceptor`

It intercepts the local updates, and it does the following:

* In the `primary-owner` generates a new `vector-clock` and stores it in the `WriteCommand` for the `backup-owners`.
* In the `backup-owners` just fetches the `vector-clock` from the `WriteCommand` and store it.

It interacts with the `IracVersionGenerator`.

### `*IracRemoteInterceptor`

It intercepts the updates from the remote site and the `primary-owners`performs the conflict resolution.

## Transactions

The transactional caches need some special handling. 
They have the following issues:

* The `vector-clock` must be generated during the `prepare` phase and not when the `WriteCommand` is executed.
* The conflicts must be detected before the transaction reaches it final outcome.
* The transaction originator may not be the `primary-onwer` for a particular `key` and
only the `primary-onwer` must generate the `vector-clock`.

### Optimistic Transactions

The Optimistic Transactions commits in 2 phases, make it easer to target the issues above. 
During the prepare phase the `primary-onwer` generates the `vector-clock` and sends it back to the originator.

The new `vector-clocks` is sent together with the `CommitCommand` to all the `backup-onwers`.

When an update from the remote site is received, it is wrapper in a transaction.
This transaction only contains a single `WriteCommand`, and the `primary-owner` checks for conflict 
during the prepare phase.
If a conflict is detected and the update needs to be discarded, it aborts the transaction.

### Pessimistic Transactions

The Pessimistic Transactions commits in one phase and, it requires a bit of work to handle IRAC properly.

The `primary-owner` acquires lock for a `key` during the transaction execution. 
So, after the lock request, the originator sends a request to obtain a new `vector-clock` for that `WriteCommand`.
It sends the request asynchronously and, it only waits for the reply, just before committing the transaction.

Conflict resolution requires a little more work. 
To avoid complicate the code dramatically, when an update from remote site is received 
it must be forwarded to the `primary owner`.
It is wrapped in the transaction and the originator (`primary-owner`) has all the data needed to check for conflicts. 

## Topology Changes

The `topology` element in the `[topology, version]` pair is here to provide consistency in case of topology changes,
and it changes every time a topology change.

When the `primary-owner` crashes, the "new" `primary-owner` will start sending the updates to the remote site. 
It may send duplicated updates for `keys` sent but yet not confirmed by the previous `primary-owner`.
With the `vector-clock`, the duplicated are detected and ignored.

Finally, the state transfer sends the pending `keys` and `tombstone` for new nodes that become owners for a segment.

