# Problem
It is possible and should be expected for a cluster to form multiple partitions (aka. [split brain](http://en.wikipedia.org/wiki/Split-brain_(computing)). E.g. if the cluster has the initial topology {A,B,C,D,E}, because of a network issue (e.g. router crash) it is possible for the cluster to split into two partitions {A,B,C} and {D,E}. Assuming DIST mode with _numOwners=2_, both partitions end up holding a subset of data and can individually be in inconsistent state.
Partitions cause inconsistencies: if in the original cluster the key _K_ is owned by nodes D and E, after the split brain the {A,B,C} partition considers _K_ as null.
At the moment Infinispan allows the users to be notified in the eventuality of a partition by registering [ViewChanged](http://docs.jboss.org/infinispan/6.0/apidocs/index.html?org/infinispan/notifications/class-use/Listener.html) listers but it doesn't offer any support for the user to react to the partition e.g. by making the cluster as inactive in order to avoid inconsistent reads. 
This wiki page documents a solution we consider for better handling of cluster partitions.

# Possible approaches
There are several approaches that can be taken for either mitigating or (eventually) solving the partitioning problem:

1. Redundant infrastructure
  * using two (or more) physical networks infrastructures the cache for partitions to happen can be reduced significantly. E.g. if the cluster is connected by two networks, each network having an availability rate of 99%, then the overall availability of the system is 99.99%. This redundancy can be configured at OS level through [IP bonding](http://www.cyberciti.biz/tips/linux-bond-or-team-multiple-network-interfaces-nic-into-single-interface.html) and doesn't require any additional Infinispan/JGroups configuration
  * note that this approach, whilst feasible for many situations doesn't entirely avoid the possibility for the partition to happen
2. Partition merging
  * in this approach the partitions can progress individually accepting read/writes from the user application (might cause inconsistencies as described above). When the two partition discover each other as a result of the network healing, the state of the two partitions is merged. There are several approaches to merge the state: e.g. automatic (would require each entry to be versioned) or pass the merging logic to the application. 
  * note that when the two partition run in parallel the data is inconsistent(AP from the [CAP](http://en.wikipedia.org/wiki/CAP_theorem) theorem)
3. Primary partition
  * both partitions make a deterministic decision on which partition to stay active and which one to go in read-only mode (or even stop serving users entirely). The decision can be made based on quorum, e.g. the partition having _numMembers/2 + 1_ nodes to win (or both to loose if a deterministic decision cannot be made, e.g. for even clusters).
  * when the network heals, the "loosing" partition can merge into the active partition (which might have been modified in between) by wiping out its state and re-fetching it (state transfer)

# Our approach
In the first iteration we plan to enhance the support for partition detection and allow the user to react to a partition happening (custom policy) by making a partition as UNAVAILABLE (stop answering users' request), READ_ONLY or AVAILABLE. This is along the lines of item 3 (Primary partition) as described in the previous section.

# Design

## New API and Config
The partition handling policy is configured in the _availability_  section of the global configuration:
```xml
  <infinispan>
       <global>
           <availability>
               <!-- this handler is invoked on every node in order to handle partitions.
                Nodes within the same partition are expected to react consistently to the partition change.
                MyPartitionStategy implements PartitionHandlingStrategy (see below).
               --> 
               <partition-handling strategy="org.acme.MyPartitionStategy"/>
               <!-- The following config is just an example and might be added in the future release in 
               order to control system's  availability when the persistent store is out...         
                < persistence-failure strategy="org.acme.MyPersistenceFailuerStrategy"/ >
               -->
           </availability>
       </global>
     ...
    </inifinispan>
```

```java
    interface PartitionContext {
        /**
         * returns the list of members before the partition happened.
        */
        View getPreviousView();
   
       /**
         * returns the list of members as seen within this partition.
        */
        View getCurrentView();
   
       /**
        * Returns true if this partition might not contain all the data present in the cluster before
        * partitioning happened. E.g. if numOwners=5 and only 3 nodes left in the other partition, then 
        * this method returns false. If 6 nodes left this method returns true.
        * Note: in future release for distributed caches, this method might do some smart computing based on 
        * segment allocations, so even if > numOwners left, this method might still return true.
        */
        boolean isDataLost();
    
        /**
        * Marks the current partition as read only (writes are rejected with an AvailabilityException).
        **/   
        void currentPartitionReadOnly();
     
        /**
        * Marks the current partition as available or not (writes are rejected with a 
        *  AvailabilityException).
        **/   
        void currentPartitionAvailable(boolean available);
    }
```

```java
    interface PartitionHandlingStrategy {
       /**
       * Implementations might query the PartitionContext in order to determine if this is the primary 
       * partition, based on  quorum and mark the partition unavailable/readonly.
       **/
       void handlePartition(PartitionContext pc);
     }
```

## Implementation details
* a new `AvailabilityInterceptor` is added, having 3 states: available, readOnly, unavailable. Based on its state the interceptor might allow, reject the writes or respectively reject all operations to the cache
* when an operation is rejected a custom exception exception is thrown to the user indicating the fact that the partition is not available (AvailabilityException)
* the `PartitionContext.currentPartitionAvailable` and `PartitionContext.currentPartitionReadOnly` methods set the state of the `AvailabilityInterceptor` and are invoked by configured `PartitionHandlingStrategy` implementation 
* the status of `AvailabilityInterceptor` is to be exposed through JMX operations as well (read/write)
* we might also provide an @Merge listener implementation to automatically merge a primary partition with an secondary (unavailable) partition by making the later wipe out it state and re-fetch it from the former. This is a useful auto-healing tool for situations where the partitioning doesn't happen because of an network error but because of e.g. a long GC on an isolated node.

## To be further considered
The partition handling strategy is intended for the whole cache manager. Wouldn't it make more sense to have a per cache strategy? E.g. a certain cache might not even be affected by a partition (e.g. if asymmetric clusters are used).

#Related
* JIRA: [ISPN-263] (https://issues.jboss.org/browse/ISPN-263)
* An interesting [mail discussion](http://infinispan.markmail.org/search/#query:+page:3+mid:qanjofgmpyvdjnmg+state:results) around the subject