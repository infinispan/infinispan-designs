On-demand operations:
```
interface ConflictResolutionManager {
   Map<Address, CacheEntry> getAllVersions(Object key); // can run from any node
   Stream<Map<Address, CacheEntry>> getConflicts(); // can run from any node, chunked?
   void resolveConflict(Object key); // applies EntryMergePolicy and updates all owners using putIfAbsent/replace/(versioned replace), always runs from primary
   void resolveConflicts();
}

// Not only for partition handling = ALLOW_ALL but also fixing failed writes
@Experimental // don't set it in stone now
interface EntryMergePolicy {
   CacheEntry merge(CacheEntry preferredEntry, CacheEntry otherEntry);
}
// OOTB implementations:
// version-based: any version is better than null, possibility to store timestamp in version to implement last update policy?
// preferred-always (return null if preferred is null)
// preferred-non-null (return preferred if none is null)

enum PartitionHandling {
   DENY_ALL, // if the partition does not have all owners, both reads and writes are allowed
   ALLOW_READS, // degraded but allows stale reads
   ALLOW_ALL // allow the values to diverge and resolve the conflicts later
}
```

Selecting preferred partition for given segment:

1. (discutable) The partition that had all owners before split?
1. Overall size of partition
1. Order in members list (just make the choice deterministic)

Selecting preferred entry when there's no split brain (failed operation kept the cluster inconsistent): preferred entry is the one from primary owner.

Automatic merging after split-brain heal:

* before the merged entry is written to all partitions, partition A should not see modifications from B and vice versa (merge happens before installing a merged topology)
* merge uses EntryMergePolicy from configuration

Configuration (<parition-handling />):

* deprecate `enabled="true|false"`
* add `type="deny-all|allow-reads|allow-all"`
* add `entry-merge-policy="version-based|preferred-always|preferred-non-null|custom class name"`

Other thoughts:

* for all-keys conflict resolution the primary needs to request whole segment, not just presently owned keys because it may miss some - we could reuse InboundTransferTask?
