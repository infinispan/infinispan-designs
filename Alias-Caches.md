In order to support materialized views, Teiid uses the following technique with relational databases:
- populates a "hidden" table with the new data
- removes the old "visible" table and renames the new "hidden" table to the "visible" name

To support this in Infinispan, instead of renaming caches, we should support Alias Caches.

## Definition
An Alias Cache is a named cache which acts as a delegate to a concrete named cache.
It is configured only with the name of a cache to which it will act as delegate. On top of the standard Cache API it also provides an additional void switchCache(String name) method with which it is possible to change the delegate. Switching is allowed only if the delegate caches mode is compatible, i.e. local <> local, clustered <> clustered. Switching between local and clustered caches is not allowed. Switching a clustered cache will be performed cluster-wide so that all aliases switch at the same time.

## Configuration
The declarative configuration, common to both embedded and server, is as follows:

`&lt;alias-cache name="alias-cache-name" delegate-cache="delegate-cache-name" /&gt;

The programmatic configuration, for embedded mode, is as follows:

`ConfigurationBuilder builder = new ConfigurationBuilder();`
`builder.aliasCache("delegate-cache-name");`
`cacheManager.defineConfiguration("alias-cache-name", builder.build());`

## Management
In embedded mode the switch operation is exposed via JMX. In server mode, the switch operation is exposed via a management operation.

