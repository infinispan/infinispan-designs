# Deployable Cache Stores high level design

This document describes Deployable Cache Stores implementation details.

## Client perspective

The client will be able to deploy a custom Cache Store jar into Hotrod server (put it into `$HOTROD_SERVER/standalone/deployments`). The jar will need to contain one of the following service entries:
* /META-INF/services/org.infinispan.persistence.spi.AdvancedCacheWriter
* /META-INF/services/org.infinispan.persistence.spi.AdvancedCacheLoader
* /META-INF/services/org.infinispan.persistence.spi.CacheLoader
* /META-INF/services/org.infinispan.persistence.spi.CacheWriter
* /META-INF/services/org.infinispan.persistence.spi.ExternalStore
* /META-INF/services/org.infinispan.persistence.spi.AdvancedLoadWriteStore

Those services might used later used in the configuration.

## Implementation details

_The implementation is based on Deployable Filters and Converters._

Currently all writers and loaders are instantiated in `PersistenceManagerImpl#createLoadersAndWriters`. This implementation will be modified to use `CacheStoreFactoryRegistry`, which will contain a list of `CacheStoreFactories`. One of the factories will be added by default - the local one (which will the same mechanism as we do now - `Util.getInstance(classAnnotation)`. Other `CacheStoreFactories` will be added after deployment scanning.

`PersistenceManagerImpl` is attached to each Cache separately and has a reference to `Configuration` object. During the startup process `PersistenceManagerImpl` scans for configured stores (`PersistenceManagerImpl#createLoadersAndWriters`) and creates separate references for `CacheLoaders` and `CacheWriters`. After modifications, not only local instances will be available (more precisely from local Classloader), but also deployed. 

## FAQ

Q: What happens when you have both a cache loader and a cache writer deployed?
A: `PersistenceManagerImpl` uses separate instances for `CacheLoaders` and `CacheWriters`. It is perfectly legal that a custom Cache Store class implements both of those interfaces. In that case the same instance will be used for loading and writing persistence data. Here is a sample block of code from, which illustrates this idea:
```
PersistenceManagerImpl#createLoadersAndWriters:
Object instance = Util.getInstance(classAnnotation);
CacheWriter writer = instance instanceof CacheWriter ? (CacheWriter) instance : null;
CacheLoader loader = instance instanceof CacheLoader ? (CacheLoader) instance : null;
...
loaders.add(loader);
writers.add(writer);
```

Q: What happens when you have a deployed Cache Store and it is also accessible locally?
A: The deployed one will be used. There are 3 options we could use here:
* throw some `DuplicatedCacheStore` exception to indicate such situation (since this is not a total disaster I wouldn't do that)
* Use the local one
* Use the deployed one (my vote)
I think using the deployed one might be useful for upgrades (for example you have a MyCustomStore v. 1.0 accessible locally and MyCustomStore v. 1.1 deployed - you would probably prefer using the newer one).

Q: How to tie a deployed cache loader/writer with the configuration itself?
A: It is already done via configuration. `PersistenceManagerImpl#createLoadersAndWriters` scans configuration and extracts Cache Store class name from:
* `ConfigurationFor#value`
* `CustomStoreConfiguration#customStoreClass`
Later on this name is used to instantiate loader/writer.

Q: Are deployed Cache Stores "active" by default?
A: Yes and no. They are accessible and can be instantiated, but without any configuration - they won't be attached to any Cache.



