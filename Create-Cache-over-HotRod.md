NB. This page refers to Hot Rod, but the concepts can be applied to REST as well.

Creating a cache from a remote client is an often requested feature:

* JCache's API CacheManager.createCache(String cacheName, C configuration)
* Hibernate OGM can optionally create caches (as part of schema generation)
* Teiid can create tables

Since cache configuration is very complex, remote cache creation will be limited to specifying an existing configuration template name. Additionally, caches created remotely might be temporary (i.e. they are not persisted back into the configuration, if applicable). The proposed API for the cache creation call (using the Java client as reference) would be:

    RemoteCache<K, V> RemoteCacheManager.createCache(String cacheName, String configurationName, boolean persistInConfiguration)

with the additional signature

    RemoteCache<K, V> RemoteCacheManager.createCache(String cacheName, String configurationName) {
        return createCache(cacheName, configurationName, false);
    }

For symmetry, a

    RemoteCacheManager.destroyCache(String cacheName)

method would remove a cache.

The protocol would be enhanced with the following opcodes:

0x37 = create cache request [header + config name length (vInt)  + config name (String) + persist (1 byte)]
0x38 = create cache response
0x39 = destroy cache request [header]
0x40 = destroy cache response

The usual status / error codes will be used in the response.

Server-side, cache creation should adhere to the following rules:

* the operation is performed on all nodes
* the operation is complete only when it is completed by all nodes
* an empty configurationName will use the configuration of the default cache
* if the temporary flag is not set, then the cache configuration will be persisted so that it will be recreated on startup
* cache creation would be disallowed on a degraded cluster
* the operation would require the ADMIN permission if authorization is enabled
* if authorization is disabled, the operation would be allowed only when invoked over a loopback interface

New nodes joining an existing cluster with caches created by the above operation might not have the required cache in their configuration. Therefore the coordinator should send back a join response with the list of running caches which need to be created, including their configuration name as well as whether they are temporary.

Persisting the configuration will only be possible if the server is in a mode which allows it:
* embedded server: N/A
* standalone mode server: the server applies the configuration to its model so that it can be persisted by the subsystem writer. If part of a cluster, the operation will need to be rolled back in case of failures on other nodes
* domain mode server: the server delegates the configuration change to its domain controller so that it is then applied to all servers in the server group. (TBD)

