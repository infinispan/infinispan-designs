# True Cache API

This design document defines a true Cache API for use cases such as Hibernate 2L cache.
Throughout the document we will use Hibernate 2L as use case for explaining rationale behind the API.


## Read-Only Cache
The most basic of cache APIs, supporting only read-only data, should allow:

1. Check if data is present in cache, if present return it, otherwise fetch it from source and store it into the cache.  
This complex operation is the essence of a caching API.
In Hibernate, this is split into two Hibernate callbacks into the cache region api: 
`CachedDomainDataAccess.get` and (maybe) `CachedDomainDataAccess.putFromLoad`.

2. Invalidate individual or all cached entries.
The user might want to invalidate individual or all cache entries in the cache. 
For example, if a fresher data has been stored into the database via methods other than Hibernate. 

4. Provide an estimate of the number of entries in cache.
A user might want to find out the estimated number of entries in a cache.
This is useful for making assertions about the cached contents.

Here's an an asynchronous API that would match these requirements:

```java
interface ReadOnlyCache<K, V> {
  
  /**
   * Checks if the cache contains a value associated with the key.
   * If the value is present, it's returned immediately wrapped in a CompletionStage.
   * 
   * If the value is not present, the function is called asynchronously to fetch the value from an unknown source.
   * If the supplier returned a value, the value is cached and then returned to the caller wrapped in a CompletionStage.
   * 
   * If the supplier didn't find a value, null is returned to the caller wrapped in a CompletionStage.
   *  
   * @param key to look up in the cache
   * @param fetcher called if not value is associated with the key in the cache
   * @return a future containing the cached entry if present, otherwise the value returned by the supplier 
   */
  CompletionStage<V> get(K key, Function<K, V> fetcher);

  /**
   * Invalidate the given key.
   * Any subsequent get calls after invalidating the key would fail to find the key in the cache. 
   * 
   * @param key to invalidate
   * @return A CompletionStage instance that signals when the invalidation has completed
   */
  CompletionStage<Void> invalidate(K key);

  /**
   * Invalidate all entries stored in the cache.
   * Any subsequent get calls would fail to find entries in the cache.
   * 
   * @return A CompletionStage instance that signals when the invalidation has completed
   */
  CompletionStage<Void> invalidateAll();
  
  /**
   * Estimated number of entries in the cache.
   * 
   * @return a CompletionStage with the estimated number of entries in the cache 
   */
  CompletionStage<Long> estimatedSize();

}
```

A non-strict implementation of this interface would not provide guarantees of results when concurrent get and invalidation occur.
So, that means that a concurrent get and invalidate might either result a cached entry or a missing entry.

As a FYI, Infinispan's implementation of Hibernate 2L offers a strict implementation of this interface.
Using a component called `PutFromLoadValidator`, it tracks the last time an entry or the cache has been invalidated.
It then compares it with the Hibernate session start time to decide which came first.
If the transaction that calls `CachedDomainDataAccess.putFromLoad` is older than the invalidation, invalidation wins.
Otherwise, `CachedDomainDataAccess.putFromLoad` is allowed to store data into the cache.
Over the years Hibernate users have come to rely on such behaviour, but such maintaining such strictness has a cost.  


## Read-Write Cache

A read-write cache supports a wider range of situations compared to a read-only cache:

It handles updates which act on both the cached value and the value at source.
It also handles removals which act both the cached value and the value at source.

On top of that, the get function can compare the cache valued and the value fetched from the source.
This is important in a read-write cache because the versions of the data in the cache and the data source might be different.
This is not the case in a read-only cache. 

```java
interface ReadWriteCache<K, V> {

  /**
   * Checks if the cache contains a value associated with the key.
   * If the value is present, it's returned immediately wrapped in a CompletionStage.
   * 
   * If the value is not present, the function is called asynchronously to fetch the value from an unknown source.
   * In the presence of concurrent updates, a value might have been cached and updated by the time the fetcher returns.
   * To handle such situations the user can provide a comparator function to help resolve discrepancies:
   * 
   * If a value exists in the cache when the fetcher completes, 
   * the comparator is called with the fetched value as first parameter,
   * and the cached value as second parameter.
   * 
   * If the comparator returns 0 or a negative number,
   * it means the fetched value is the same or older than the cached value,
   * in which case the fetched value is not cached and the cached value is returned wrapped in a CompletionStage.
   * 
   * If the comparator returns a positive number,
   * the fetched value is newer than the cached value,
   * in which case the fetched value is cached and returned wrapped in a CompletionStage.
   * 
   * @param key to look up in the cache
   * @param fetcher called if not value is associated with the key in the cache
   * @param comparator called if a cached value exists when the fetcher returns 
   * @return a future containing the cached value or null if not values exist 
   */
  CompletionStage<V> get(K key, Function<K, V> fetcher, Comparator<V> comparator);

  /**
   * Remove entry from the cache and remove it from source by calling the remover function.
   * 
   * @param key to delete from cache and source
   * @param remover function that deletes data from source
   * @return a future that is completed when key has been removed from both cache and source  
   */
  CompletionStage<Void> remove(K key, Consumer<K> remover);
  
  /**
   * Update the cached value associated with the key,
   * and update the source by calling the updater function.
   *  
   * @param key to update in cache and source
   * @param value new value for the key
   * @param updater function that updates the value associated with the key in the source
   * @return a future that is completed once the key has been updated in both cache and source 
   */
  CompletionStage<Void> update(K key, V value, BiConsumer<K, V> updater);

  // Same as ReadOnlyCache
  CompletionStage<Void> invalidate(K key);
  
  // Same as ReadOnlyCache
  CompletionStage<Void> invalidateAll();

  // Same as ReadOnlyCache
  CompletionStage<Long> estimatedSize();
}
```

By handing over the control of executing the update/removal at source to the cache,
cache implementations can decide whether to have a strict or non-strict implementation:


### Non-Strict Read-Write Cache Implementation

In the presence of concurrent `ReadWriteCache.get` and `ReadWriteCache.remove` calls,
implementations are free to end up caching value before remove, or not caching any value at all.

In the presence of concurrent `ReadWriteCache.get`, `ReadWriteCache.invalidate` and `ReadWriteCache.update` calls,
implementations are free to cache the value after update, the value before update or no value at all.  


### Strict Read-Write Cache Implementation

A strict implementation would be akin to Infinispan implementation of Hibernate 2L cache.
Using techniques similar to the `PutFromLoadValidator` mentioned in earlier section, 
it can provide more strict guarantees based on timestamps (or version information?)  


#### Concurrent `ReadWriteCache.get` and `ReadWriteCache.remove` 

Strict implementations would decide that after removing the entry from the cache,
they would not allow `ReadWriteCache.get` calls that fail to find the entry to call to the `Supplier<V>` fetcher function.
Once the function that deletes the entry from the DB would complete, `ReadWriteCache.get` would behave as normal. 
 
In Hibernate terms, the problem that is being handled is the following:

Thread-1: read(A) from DB, calls putFromLoad and write(A) to cache
Thread-1: delete(A) from cache
Thread-2: read(A) from cache returns null
Thread-2: read(A) from DB, calls putFromLoad and write(A) to cache
Thread-1: delete(A) from DB
Result: Cache ends up with deleted data (A)

As a FYI, the Infinispan implementation of Hibernate 2L handles this situation using the `PutFromLoadValidator`:

Thread-1: read(A) from DB, calls putFromLoad, success acquiring pFL lock, write(A) to cache.
Thread-1: delete(A) from cache and invalidate putFromLoad calls for A until tx after completion [*].
Thread-2: read(A) from cache returns null
Thread-2: read(A) from DB, calls putFromLoad and can’t acquire pFL due to invalidation
Thread-1: delete(A) from DB
Result: Cache ends up with no data
[*] Before invalidating putFromLoad for a given key, a javax.transaction.Synchronization is registered that on afterCompletion ends the invalidation and entries can be cached again.


### Concurrent `ReadWriteCache.get`, `ReadWriteCache.invalidate` and `ReadWriteCache.update`

Strict implementations would decide that if any invalidation happened after updating the entry from the cache,
they would not allow `ReadWriteCache.get` calls that fail to find the entry to call to the `Supplier<V>` fetcher function.
Once the function that updates the entry from the DB would complete, `ReadWriteCache.get` would behave as normal. 

In Hibernate terms, the problem that is being handled is the following:

Thread-1: insert(A,V1) to DB
Thread-1: write(A,V1) to cache
Thread-2: read(A) from cache returns (A,V1)
Thread-2: update(A, V2) in cache
Thread-3: invalidate cache
Thread-3: read(A) from cache returns null
Thread-3: read(A) from DB returns V1, calls putFromLoad and writes(A, V1) to cache
Thread-2: update(A, V2) in DB
Result: Cache contains V1 instead of V2

For reference, the Infinispan implementation of Hibernate 2L handles this situation using the `PutFromLoadValidator`: 

Thread-1: insert(A,V1) to DB
Thread-1: write(A,V1) to cache
Thread-2: read(A) from cache returns (A,V1)
Thread-2: update(A, V2) in cache and invalidate putFromLoad calls for A until after completion
Thread-3: invalidate cache
Thread-3: read(A) from cache returns null
Thread-3: read(A) from DB returns V1, calls putFromLoad and can’t acquire pFL due to invalidation
Thread-2: update(A, V2) in DB
Thread-2: after completion called
Result: Cache does not contain A


Configuration
-------------

Regardless of whether a read-only or read-write cache is in use, caches should be configured with eviction by default.
It should still be possible to disable eviction when caching small data.
For example, Hibernate's update timestamps is a type of cache where eviction should be disabled.
