Currently the ClusterRegistry provides some syntactic sugar around a replicated cache which can be used as an internal repository for service-data.
To differentiate between different scopes and uses, the ClusterRegistry uses a composite key composed of a scope and a key.
While useful, the current design is very limited:
* it doesn't allow configuration of the underlying replicated cache (i.e. persistence, security, eviction, etc)
* some use cases might prefer a more efficient invalidation approach instead of a replicated one
* the wrapping of the key adds an extra object and it makes it cumbersome to expose this cache to users (Remote query would need to)
* many internals avoid using preferring custom solutions (e.g. Map/Reduce temporary caches, Remote Query schema cache)

The new ClusterRegistry should behave as follows:
* create a dedicated per-service cache instead of a single catch-all cache
* the characteristics of a registry cache are specified by the requiring service
* persistent/volatile
* replication/invalidation
* eviction/expiration
* security
* naming strategy (i.e. static name, derived name, etc)
* dependencies (so that a registry cache's lifecycle is bound to another cache)
* custom settings (Remote Query schema cache needs a custom interceptor)

Potential users of registry caches
* Remote Query schema cache
* Query index caches
* Map/Reduce temporary caches
* Security ACL cache
* ...

