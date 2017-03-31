Next major Infinispan version will be based on Java 8 which brings a lot of new functional programming features. This document describes a proposal for Infinispan to make the most of the Java 8 features and indirectly enable new use cases for Infinispan.

# Document history
13/03/2015 - First draft

# Background

This section focuses on providing background information on the Java 8 API proposal.

## What's wrong with the Infinispan Cache API?

The biggest problem of the Infinispan Cache API is that it mixes the API that is exposed externally (`org.infinispan.Cache` and indirectly `java.util.concurrent.ConcurrentMap`), and the API that Infinispan needs to be able to implement to be able to fulfil the contracts that are exposed to the user. As a result of this, it's difficult to add new operations to Infinispan, or expose new contracts (e.g. distributed queues), without causing major havoc in the Infinispan internals.

## How does Java 8 improve on this?

Java 8 enables functions/lambdas to be passed to methods, enabling behaviour to be passed in Infinispan. As a result of this, a core, distilled, functional cache API can be created that taking functions, can implement all of the operations exposed externally to the user. Such functional API would contain a subset of operations that enable Infinispan to expose multiple contracts such as `java.util.Map`, `java.util.concurrent.ConcurrentMap` or `javax.cache.Cache`.

Apart from being able to have a distilled caching API for existing map-like data structures, Java 8 enables other data structures to more efficiently be implemented. For example, if exposing a numeric counter API, we could add two operations `increment` and `decrement` which work on a cached value, the first adding `1` to the value, and the second subtracting `1` from the value. In both cases, adding/substracting would be part of the behaviour passed in via lambda.

A key factor here is that Java 8 functions/lambdas can be distributed if the functional interfaces are made to extend Serializable, so we can actually distribute the behaviour. This is very important when trying to implement higher level data structures on top of Infinispan. For example, if adding to a queue, you'd only need to replicate the function that adds an element to the queue, rather than replicating the entire queue contents. In other words, Java 8 enables Infinispan to implement delta-aware data structures the proper way.

# Proposal

This section focuses on what this Functional Cache API should look like to implement the operations exposed by Infinispan. The objective is for the Functional Cache API to be able to be used to expose the main contracts currently exposed by Infinispan including:
* `java.util.concurrent.ConcurrentMap` (and by extension `java.util.Map`)
* `javax.cache.Cache`

The Functional Cache API should, as much as possible, allow for all operations exposed by `org.infinispan.AdvancedCache` to be exposed. Some of those though are rather SPI like operations, e.g. component registry related operations, and how to expose those in the future is TBD.

This Functional Cache API has been prototyped and it has been verified that it can be used to implement the `java.util.Map` interface (see 'Proposal In Action' section).

**NOTE**: More methods will need adding to be able to fully expose all the contracts currently exposed. Also, some details have been removed from this API to make it more easily readable. The full API is available in the link provided in the prototype section.

At a glance, all the methods in this interface return `CompletableFuture<>`. The main reason for doing so is because for the big majority of use cases Infinispan is a distributed data structure, hence the vast majority of operations require to go remote to another node in the cluster or remote to some sort of persistence layer. Returning Java 8's CompletableFuture enables clients to behave more reactively, e.g. when the operation has completed, do the following..., instead of blocking for operations. 

Even though we expose `CompletableFuture<>` instances externally, we will work to provide ways to avoid thread creation and context switching if the user will wait for the CompletableFuture<> to complete immediately. This can easily be done using tricks like the ones used by Infinispan's Flags.

```java
/** 
 * A functional cache is a distilled caching interface that enables functional,
 * behaviour-based, manipulation of the contents of the cache.
 */
public interface FuntionalCache<K, V> {

   /**
    * Evaluates a function against a given key. The function takes a Value
    * instance as parameter, which is a place-holder for the current value
    * associated with that key in the cache. The function then returns an
    * instance of a type defined for the user.
    *
    * With this method, a very big subset of operations can be implemented.
    * For example: Map's get/put/containsKey/remove, ConcurrentMap's
    * putIfAbsent/replace/remove, JCache's getAndPut... and many others.
    *
    * AccessMode is an crucial parameter which provides a hint on what the
    * function will be doing. For example, if access mode is read only,
    * then it means that we don't need to acquire locks on the key. If the
    * access mode is write only, then we don't need to retrieve the current value
    * associated with the key.
    *
    * If the function does not return any value, the method should expect a `Void`
    * return instead of `void`. This avoids the need to overload this method to 
    * take a Consumer lambda. This is an unfortunate side effect of Java's design 
    * decision not to treat `void` as an Object.
    */
   <T> CompletableFuture<T> eval(K key, AccessMode mode, 
         ValueFunction<? super V, ? extends T> f);

   /**
    * Evaluates a function against a collection of key/value pairs. The
    * function is applied for each key in the collection, and the function
    * takes the value passed in associated with the key, and a place-holder for
    * for the current value associated with that key in the cache.
    *
    * The function signature has been designed in such way into order to make
    * sure that it takes the value to be associated with that cache entry from
    * the map passed in. Alternative designs could involve passing in the key
    * and the function itself looking up the entry in the map, but doing so
    * means that each function references the full map and hence it'd bloat the
    * function considerably when trying to serialize it.
    *
    * It is Ok for single key functions to reference an external value because
    * it's only one entry involved, but when you have multiple entries, each
    * function should only care about the value it needs, it doesn't need to
    * know about all values.
    *
    * This method can be used to implement both Map's putAll and, with a
    * little bit of tweaking, JCache's getAll.
    *
    * The reason why this method is so important, instead of letting the user
    * iterate and call eval(K) is because internal components can decide how
    * the execution should be done and do any necessary splitting to be as
    * efficient as possible.
    */
   <T> Map<K, CompletableFuture<T>> evalAll(Map<? extends K, ? extends V> iter, AccessMode mode,
         ValueBiFunction<? super V, ? extends T> f);

   /**
    * Clears the cache, an operation that works on the cache as a whole.
    * It could be implemented via `evalAll`, but implementations often have 
    * a way to execute it in a much more efficient way than iterating over 
    * each entry, hence keep it as a separate method.
    */
   CompletableFuture<Void> clearAll();

   // IMPORTANT: 
   // FunctionalCache should not expose Stream instances externally because Stream 
   // methods are designed for collections that are not concurrently updated. 
   // Instead, we should select those functions that are most relevant for the end-user
   // use cases. This is exact same reason why ConcurrentHashMap does not expose
   // any Streams directly.

   /**
    * Search function, return the first available non-null result of applying
    * a given function on each element; skipping further search when a result
    * is found.
    *
    * The search mode defines the scope of the search, whether keys only,
    * values only, or both. If keys only are searched, value parameter is always null.
    * If values only searched, keys is null.
    *
    * To avoid bloating the API with Optionals, and avoid the need to overload for
    * different search types, Optional is reserved only for the return parameter,
    * when the user can type safely determine whether the search found something
    * or not.
    */
   <T> CompletableFuture<Optional<T>> search(StreamMode mode,
         PairFunction<? super K, ? super V, ? extends T> f);

   /**
    * Apply a fold function on all entries (e.g. can be used to calculate size).
    * It differs with search in that the fold function gets applied to all
    * elements in the cache, whereas search stops the moment the function returns
    * non-null.
    */
   <T> CompletableFuture<T> fold(StreamMode mode, T z,
         PairBiFunction<? super K, ? super V, ? super T, ? extends T> f);
}

@FunctionalInterface
public interface ValueFunction<V, T> extends Function<Value<V>, T>, Serializable {}

@FunctionalInterface
public interface ValueBiFunction<V, T> extends BiFunction<V, Value<V>, T>, Serializable {}

@FunctionalInterface
public interface PairFunction<K, V, T> extends Function<Pair<K, V>, T>, Serializable  {}

@FunctionalInterface
public interface PairBiFunction<K, V, T1, T2> extends BiFunction<Pair<K, V>, T1, T2>, Serializable  {}

public interface Value<V> {
   /**
    * Optional value. It'll return a non-empty value when the value is present,
    * and empty when the value is not present.
    */
   Optional<V> get();

   /**
    * Set this value.
    * 
    * It returns 'Void' instead of 'void' in order to avoid, as much as possible, 
    * the need to add `Consumer` overloaded methods in FunctionalCache.
    */
   Void set(V value);

   /**
    * Removes the value.
    * 
    * Instead of creating set(Optional<V>...), add a simple remove() method that
    * removes the value. This feels cleaner and less cumbersome than having to 
    * always pass in Optional to set()
    */
   Void remove();
}

public static enum AccessMode {
  READ_ONLY, READ_WRITE, WRITE_ONLY;
}

public static enum StreamMode {
   KEYS_ONLY, VALUES_ONLY, KEYS_AND_VALUES
}

/**
 * Represents an immutable key/value pair.
 *
 * This class has been designed for use with functional cache stream-like methods (search and fold).
 */
public interface Pair<K, V> {
   /**
    * Optional key.
    * It'll be empty when only iterating over cached values.
    * It'll be non-empty when iterating over keys or both keys and values.
    */
   Optional<K> key();

   /**
    * Optional value.
    * It'll be empty when only iterating over cached keys.
    * It'll be non-empty when iterating over values or both keys and values.
    */
   Optional<V> value();
}
```

# Proposal In Action

The following code shows how the FunctionalCache API can be used to implement the java.util.Map:

```java
private static class MapDecorator<K, V> implements java.util.Map<K, V> {
   final FuntionalCache<K, V> cache;

   private MapDecorator(FuntionalCache<K, V> cache) {
      this.cache = cache;
   }

   @Override
   public int size() {
      // FIXME: Could be more efficient with a potential StreamMode.NONE
      return await(cache.fold(StreamMode.KEYS_ONLY, 0, (p, t) -> t + 1));
   }

   @Override
   public boolean isEmpty() {
      // Finishes early, as soon as an entry is found
      return !await(cache.search(StreamMode.VALUES_ONLY,
            (p) -> p.value().isPresent() ? true : null)
      ).isPresent();
   }

   @Override
   public boolean containsKey(Object key) {
      return await(cache.eval(toK(key), READ_ONLY, e -> e.get().isPresent()));
   }

   @Override
   public boolean containsValue(Object value) {
      // Finishes early, as soon as the value is found
      return await(cache.search(StreamMode.VALUES_ONLY,
         (p) -> p.value().get().equals(value) ? true : null)
      ).isPresent();
   }

   @Override
   public V get(Object key) {
      return await(cache.eval(toK(key), READ_ONLY, e -> e.get().orElse(null)));
   }

   @SuppressWarnings("unchecked")
   private K toK(Object key) {
      return (K) key;
   }

   @Override
   public V put(K key, V value) {
      return await(cache.eval(toK(key), READ_WRITE, v -> {
         V prev = v.get().orElse(null);
         v.set(value);
         return prev;
      }));
   }

   @Override
   public V remove(Object key) {
      return await(cache.eval(toK(key), READ_WRITE, v -> {
         V prev = v.get().orElse(null);
         v.remove();
         return prev;
      }));
   }

   @Override
   public void putAll(Map<? extends K, ? extends V> m) {
      Map<K, CompletableFuture<Object>> futures = cache.evalAll(m, WRITE_ONLY, (x, v) -> {
         v.set(x);
         return null;
      });

      // Wait for all futures to complete
      await(CompletableFuture.allOf(
         futures.values().toArray(new CompletableFuture[futures.size()])));
   }

   @Override
   public void clear() {
      await(cache.clearAll());
   }

   @Override
   public Set<K> keySet() {
      return await(cache.fold(StreamMode.KEYS_ONLY, new HashSet<>(), (p, set) -> {
         set.add(p.key().get());
         return set;
      }));
   }

   @Override
   public Collection<V> values() {
      return await(cache.fold(StreamMode.VALUES_ONLY, new HashSet<>(), (p, set) -> {
         set.add(p.value().get());
         return set;
      }));
   }

   @Override
   public Set<Entry<K, V>> entrySet() {
      return await(cache.fold(StreamMode.KEYS_AND_VALUES, new HashSet<>(), (p, set) -> {
         set.add(new Entry<K, V>() {
            @Override
            public K getKey() {
               return p.key().get();
            }

            @Override
            public V getValue() {
               return p.value().get();
            }

            @Override
            public V setValue(V value) {
               V prev = p.value().get();
               cache.eval(p.key().get(), WRITE_ONLY, v -> v.set(value));
               return prev;
            }

            @Override
            public boolean equals(Object o) {
               if (o == this)
                  return true;
               if (o instanceof Map.Entry) {
                  Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                  if (Objects.equals(p.key().get(), e.getKey()) &&
                     Objects.equals(p.value().get(), e.getValue()))
                     return true;
               }
               return false;
            }

            @Override
            public int hashCode() {
               return p.hashCode();
            }
         });
         return set;
      }));
   }
}

public static <T> T await(CompletableFuture<T> cf) {
   try {
      return cf.get();
   } catch (InterruptedException | ExecutionException e) {
      throw new CacheException(e);
   }
}
```

# Decorators, decorators, decorators

With the creation of `FunctionalCache`, we can finally separate what is it that Infinispan needs to implement to do its job, versus what is that we want to expose externally. So, once we have FunctionalCache, we will create a set of decorators for `java.util.Map`, `java.util.concurrent.ConcurrentMap` and `javax.cache.Cache`.

Down the line, we'll be able to provide more decorators to keep counters in memory, implement distributed queues...etc.

# Minimising disruption

To start with, Infinispan's Cache API should still be usable, and its internal operations should continue to use the existing visitor architecture. Adding new API and changing the internals to only use new functional cache related visitor commands would be too much work to do in one go.

# FAQs

## What happens with `org.infinispan.Cache` API?

Deprecate but still usable in Infinispan 8. Functional cache must be able to provide most of the functionality. If there's any method that cannot be provided by functional cache, it should be noted.

## What happens with `org.infinispan.AdvancedCache` API?

Should be deprecated and find a better place for the SPI methods that expose internals. Right now, it mixes both internal SPI-like operations and caching operations recently (e.g. metadata based cache operations).

## What happens with Infinispan's Flags?

We need better flag system that enables operations to be tweaked. Enums are still good for this but they need to be tweaked. Firstly, there needs to be a divide between internal and external flags. Then, we need to have flags that enable values to be passed in along with the flag, e.g. to be able to pass in things like expiration values.

## How will you provide per-invocation expiration?

New flag system should support value-bearing flags. We should also get rid of max idle expiration since it's very problematic and it's essentially broken (e.g. cache entry last update time not updated in persistent stores or in other nodes in the cluster, and doing so would slow things down a lot).

We want to avoid overloading methods as much as possible in FunctionalCache because it pollutes the API, hence why flags should provide a way to pass in optional parameters such as expiration.

## Are there any performance concerns?

Some of the most performance critical caching operations might be defined as they are within FunctionalCache, e.g. `get()`, but any such operations would only be implemented if strictly required from a performance perspective.

Using functional cache APIs might have a slight increase in cost compared to the standard APIs, but down the line, they'll enable some interesting optimisations. For example:

Given a Counter API, which exposes `increment` and `decrement` operations, the functions used to increment and decrement are `Commutative`, so this means that they can be executed in any other and the result will be the same. If the user could provide us with a flag indicating that the operation is commutative, we could apply both write functions in parallel.

## Are there any dangers/pitfalls in this funtional cache API?

Lambda's could capture big objects, which would affect serialization speed. Also, captured objects could be non-Serializable and hence it'd break at runtime. However, users not forced to use FunctionalCache, they can use the decorators to avoid the need to enter the functional cache space, so the risk is low.

## What happens with RemoteCache?

RemoteCache API needs a rethink as well because a lot of the methods it exposes do not work well in a remote environment, such as value-based comparisons, key/value/entry sets...etc. This API is likely to be different to the functional proposal above because whereas in embedded you'd pass a Java Lambda, in remote, you might want to execute something that's more portable language wise, e.g. Javascript.

# Proposal prototype

I've created a [java8](https://github.com/galderz/infinispan/tree/z_java8) branch containing the prototype done to study the viability of the API exposed. This [link](https://github.com/infinispan/infinispan/compare/master...galderz:z_java8) provides an overall view of all the code changes related to the prototype.