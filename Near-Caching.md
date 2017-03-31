The scope of Infinispan Near Caches is to add optional L1 caching to Hot Rod client implementations in order to keep recently accessed data close to the user. This L1 cache is essentially a local Hot Rod client cache that gets updated whenever an a remote entry is retrieved via `get` or `getVersioned` operations. By default, Hot Rod clients have near caching disabled.

Near Cache consistency is achieved with the help of remote events, which send notifications to clients when entries are modified or removed. At the client level, near caching can be configured to either be Lazy or Eager. When enabling near caching, the user must decide between these two modes.

# Lazy Near Caches

Entries are only added to the lazy near caches when they are retrieved remotely via `get` or `getVersioned`. If the cache entry is modified or removed server side, the Hot Rod client receive events which in turn invalidate the near cache entries by removing them from the near cache. This way of keeping near cache consistent is very efficient because the events sent back to the client only contain key information. The downside is that if a cache entry is retrieved after it's been modified, the Hot Rod client will have to fetch it from the remote server.

# Eager Near Caches

Eager near caches are eagerly populated as entries are created in the server, and when entries are modified, the latest value is sent along with the notification to the client and it stores it in the near cache. Eager caches, same as lazy near caches, are also populated when an entry is retrieved remotely (if not already present). The advantage of eager near caches is that it can reduce the number of trips to the server by having newly created entries already present in the near cache before any requests to retrieve it have been received. Also, if modified entries are re-queried by the client, these can be fetched directly from the near cache. The disadvantage of eager near caching is that events received from the server bigger in size due to the need to ship back value information, and entries could be sent to client that will never be queried.

To limit the cost the eager near caching, created events are only received for those entries that are created after the client has connected to the Hot Rod servers. Existing cached entries will only be added to the near cache if the user queries them

## Filtering

Some of the downsides of eager near caches could be mitigated by enabling users to add filtering of near cache entries. In other words, the users could potentially in the future provide a filter implementation that defines which keys to fetch eagerly. This filter could be used to enable existing cached entries to be sent back to the client when the client connects to the Hot Rod servers, and could also be used to filter which created events to send back once it's already connected. **This feature is not currently planned for inclusion**

# Eviction

Being able to keep control the size of near caches is important. Even though not enabled by default, eviction of near caches can be configured by defining the maximum number of elements to keep in the near cache. When eviction is enabled, an LRU LinkedHashMap is used (protected by a `ReentrantReadWrite` lock to deal with concurrent updates). A better solution based around a BoundedCHMv8 is planned.

# Cluster

Near caches are implemented using Hot Rod remote events, which underneath use cluster listeners as technology for receiving events across the cluster. A key characteristic of cluster listeners is that within a cluster they are installed in a single node, with the rest of nodes sending events to the this node. So, it's possible for node that runs the near cache backing cluster listener to go down. In such situation, another node takes over running the cluster listener. When such thing happens, a client failover event callback can be defined to be invoked. For near caches, such callback will be implemented and its implementation will consist of clearing the near cache. Clearing is the simplest and most efficient thing to do, since during the failover events might have been missed.