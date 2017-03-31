We should provide a client library which is capable of performing remote administration operations using the DMR.
This would be used in a number of places: as a general remote admin client, as a helper for remote JCache, to allow Teiid to perform alias switching in remote configurations.

## Functionality
The client should provide a subset of management operations which are available within a server:

* creation/removal of caches
* creation/removal of cache configurations
* cache lifecycle management (start/stop)
* switching cache aliases
* etc

## API
If possible we should reuse the embedded configuration API.

## Security
The client will use the security policies enabled in the management security domain of the server.

## Packaging
The remote admin client should be packaged as a separate jar from the infinispan-remote jar: infinispan-remote-admin-$VERSION.jar. This uberjar would contain the necessary jboss client, xnio and other dependencies required to communicate with the admin endpoint.