#Context
The current(as of Infinispan 6.0) eventing support in Infinispan has the following limitation: the registered listeners are local to the node where the event has produced. E.g. considering an distributed cache with _numOwners=2_ which runs on nodes {A,B,C,D,E} and _K_ being a key in the cache, with owners(K)={A,B}. A [@CacheListener](http://docs.jboss.org/infinispan/6.0/apidocs/org/infinispan/server/websocket/CacheListener.html) instance registered on node C does not receive notifications on any events related to the key _K_ such as the entry being created, removed or updated. It is only the **local** listeners in A and B which receive this notifications.
 
This wiki page describes an design approach for implementing **cluster** listeners: listeners that are to be notified on certain events disregarding where the events actually occurred. Resuming the previous example, a **cluster** listener registered on C can subscribe to notifications of events originated by adding, removing or updating _K_.

## Befits
There are several problems that **clustered** listeners help solving:
* [materialized views] (http://en.wikipedia.org/wiki/Materialized_view): for queries that are run very frequently against the grid (e.g. few k times a second) instead of repeating the query on every request, one can register a cluster listener that defines the query and keep track of the query result. This fits very nicely for situations where the query is run frequently but the result of the query is amended rarely. 
* [Complex Event Process (CEP)] (http://en.wikipedia.org/wiki/Complex_event_processing) because the listeners allow keeping state between invocations, certain basic CEP conditions can be implemented within the listeners. For more complex state processing (or in order be allowed to define CEP logic dynamically) an proper CEP engine can be plugged in, such as [Drools](http://www.jboss.org/drools/)

#Suggested solution
For consistency and simplicity we plan to build the cluster listener logic on top of the existing  [@CacheListener](http://docs.jboss.org/infinispan/6.0/apidocs/org/infinispan/server/websocket/CacheListener.html) API. 

##API changes 

The existing `Cache` API adds an overloaded `addListener` method with the following signature: 
```java
/**
* Registers a listener that will be notified on events that pass the filter condition.
* @param filter decides which events should be feed to the listener. Particularly relevant for cluster 
* listeners as it reduces the network traffic by only propagating the relevant events to the node where the 
* listener is registered.
* @param converter creates a projection on the filtered entry's value which is then returned by the 
* CacheEntryCreatedEvent#getValue(). Particularly relevant for clustered listeners in order to reduce the 
* size of the data sent by the event originating node to the event consuming node (where the listener is 
* located). 
*/
Cache.addListener(Object listener, Filter filter, Converter converter);
```

The listener annotation is enhanced with two new attributes:
```java
public @interface Listener {
  /**
  * Defines whether the annotated listener is clustered or not. 
  * Important: Clustered listener can only be notified for @CacheEntryRemoved, @CacheEntryCreated and   
  * CacheEntryModified events.
  */
  boolean clustered() default false;

  /**
  * For clustered listeners only: if set to true then the entire existing state within the cluster is 
  * evaluated. For existing matches of the value, an entryAdded events is triggered against the listener    
  * during registration.  
  **/
  boolean includeCurrentState() default false;
}
```

```java
/**
* Accepts or rejects an entry based on key, value and metadata. In the case of entries being removed, the 
* value and the metadata are null. 
*/
interface Filter<K,V> {
  boolean accept(K key, V value, Metadata metadata)
}
```

```java
/**
* Builds a projection of V which is send to the node where the listener is registered.
*/
interface Convertor<K,V,C> {
  C convert(K,V, Metadata metadata);
}
```

##Lifecycle
The (cluster) listener can be registered with the `Cache.addListener` method described above and is active till one of the following two events happen:
* it is explicitly unregistered by invoking `Cache.removeListener`
* the node on which the listener was registered crashes

##Guarantees
###Ordering
For cluster listeners, the the order in which the cache is updates is reflected in the sequence of notifications received.
###Singularity
The cluster listener does not guarantee that an event is sent once and only once. Implementors should guard agains a situation in which the same event is send twice or multiple times, i.e. the listener implementation should be idempotent. Implementors can expect singularity to be honored for stable clusters and outside of the time span in which synthetic events are generated as a result of _includeCurrentState_.

##Internals
The sequence diagram below describes how we plan to implement listeners. 
![Generated with Astah community](https://lh5.googleusercontent.com/na8Tm1WbHyhyQm0tnydnyFXBSo5mFHigg2mWdirFgcoF5JlkibOpbYE_7ru_Np3mNrb0cNguVUM=w1256-h898)
* 1.1: the filter attached to the listener is broadcasted to all the nodes in the cluster. Each node holds a structure Filter -> {Node1, NodeX} mapping the set of nodes interested in events matching the given filter
* 1.1.1.checkFilterAlreadyRegistered is a potential optimization for the case in which the same filter is used by multiple listeners within the cluster
* 2: for each write request, the the registered filters are invoked and the corresponding remote listener(s) are notified

###Handling includeCurrentState=true
* in parallel with 1.1.1 a Map/Reduce command is run against the cluster that would feed @CacheEntryCreated events into the listener instance
* after 1.1.1 completes, for every operation matching the filter, instead of 2.1.1:notifyRemoteFilterOnMatch, the node keeps a (bounded) queue of the modifications for future processing
* after the Map/Reduce task is finalized the node where the listener is installed broadcasts a message to the cluster allowing the queue messages to be flushed
* this approach guarantees ordering

###Filter housekeeping
If a node crashes, then the filters originated from that node are removed from the list (assuming the filter is not in use by listeners registered on other nodes). This can be implemented using a ViewChangeListener.

###Handling topology changes
Cluster registration information should be propagated as part of the state transfer, for new joining nodes. This would be an additional step in the state transfer process, together with the memory state and the persistent state. Other functionality might need to transfer state as well (e.g. remote listeners to transfer the remote listener information) so we might want to make this a pluggable mechanism.

#Related
JIRA: [ISPN-3355] (https://issues.jboss.org/browse/ISPN-3355)
A relevant [discussion](http://markmail.org/thread/qanjofgmpyvdjnmg) on the mailing list is (even though interlaced with the partitioning one).