# Multimap Enhancements Design

## Tracking Issue

https://issues.redhat.com/browse/ISPN-13853

Replaces the previous design: [Multimap-As-A-First-Class-Data-Structure](https://github.com/infinispan/infinispan-designs/blob/main/Multimap-As-A-First-Class-Data-Structure.asciidoc)

## 1. Collection Types

Multimap values are collections. The current API treats all multimaps as generic bags with an optional `supportsDuplicates` flag. Instead, the collection semantics should be explicit and determine which operations are available.

### Collection type matrix

| Type | Ordering | Duplicates | Access pattern |
|------|----------|------------|----------------|
| **SET** | None | No | By value |
| **BAG** | None | Yes | By value |
| **LIST** | Insertion order | Yes | By value or index |
| **SORTED_SET** | Score-based | No | By value, score, or rank |
| **MAP** | None | No (by inner key) | By inner key |
| **ARRAY** *(future)* | By long index | No (by index) | By index, sparse |

The first five map directly to the existing internal bucket implementations: `SetBucket`, `Bucket`, `ListBucket`, `SortedSetBucket`, and `HashMapBucket`. ARRAY would require a new bucket type — essentially a `Map<Long, V>` that supports sparse indexing with gaps, unlike LIST which is dense and insertion-ordered.

## 2. Declarative Configuration

Multimaps are built on top of caches. Rather than creating top-level XML elements for every combination of cache mode and collection type (e.g. `replicated-list`, `distributed-list`), the collection type is declared via a namespace extension on the cache element. This avoids combinatorial explosion and keeps the configuration schema maintainable.

The multimap namespace elements are a closed set — one element per collection type, no attributes, mutually exclusive within a cache definition:

```xml
<!-- SET: unordered, no duplicates -->
<distributed-cache name="myset">
   <multimap:set/>
</distributed-cache>

<!-- BAG: unordered, allows duplicates -->
<distributed-cache name="mybag">
   <multimap:bag/>
</distributed-cache>

<!-- LIST: insertion-ordered, allows duplicates, index access -->
<replicated-cache name="mylist">
   <multimap:list/>
</replicated-cache>

<!-- SORTED_SET: score-ordered, no duplicates, rank/range access -->
<distributed-cache name="myscores">
   <multimap:sorted-set/>
</distributed-cache>

<!-- MAP: inner key-value pairs -->
<distributed-cache name="mymap">
   <multimap:map/>
</distributed-cache>

<!-- ARRAY (future): sparse, indexed by long position -->
<distributed-cache name="myarray">
   <multimap:array/>
</distributed-cache>
```

For queryable multimaps, the value type comes from the existing `<indexing>` configuration — no new elements needed:

```xml
<distributed-cache name="indexed-tags">
   <encoding media-type="application/x-protostream"/>
   <indexing enabled="true">
      <indexed-entities>
         <indexed-entity>Tag</indexed-entity>
      </indexed-entities>
   </indexing>
   <multimap:set/>
</distributed-cache>
```

The XML schema enforces correctness structurally: only one `multimap:*` element is valid per cache, and the element name fully determines the collection semantics.

## 3. API Module Redesign

The `api` module interfaces (`SyncMultimap`, `AsyncMultimap`) are still `@Experimental` and can be changed freely. The legacy `MultimapCache` in the `multimap` module stays as-is.

### Configuration

```java
interface MultimapConfiguration {
   CollectionType collectionType(); // SET, LIST, SORTED_SET, MAP, BAG
}
```

```java
enum CollectionType {
   SET,
   BAG,
   LIST,
   SORTED_SET,
   MAP
}
```

### Base interface (common to all collection types)

```java
interface SyncMultimap<K, V> {
   String name();
   MultimapConfiguration configuration();
   SyncContainer container();

   void add(K key, V value);
   CloseableIterable<V> get(K key);
   boolean remove(K key);
   boolean remove(K key, V value);
   boolean containsKey(K key);
   boolean containsEntry(K key, V value);
   long estimateSize();

   <T> Query<T> query(String ickle);
}
```

### Type-specific extensions

#### SyncMultimapList

Adds positional access for ordered, duplicate-allowing collections.

```java
interface SyncMultimapList<K, V> extends SyncMultimap<K, V> {
   V get(K key, long index);
   void add(K key, long index, V value);
   V set(K key, long index, V value);
   V remove(K key, long index);
   long indexOf(K key, V value);
   CloseableIterable<V> subList(K key, long from, long to);
}
```

#### SyncMultimapSortedSet

Adds score-based and rank-based access.

```java
interface SyncMultimapSortedSet<K, V> extends SyncMultimap<K, V> {
   CloseableIterable<V> range(K key, double minScore, double maxScore);
   Long rank(K key, V value);
   Double score(K key, V value);
}
```

#### SyncMultimapMap

Adds inner-key access for nested key-value pairs. The third type parameter `HK` represents the inner (hash) key.

```java
interface SyncMultimapMap<K, HK, HV> extends SyncMultimap<K, HV> {
   HV get(K key, HK hashKey);
   void put(K key, HK hashKey, HV value);
   boolean remove(K key, HK hashKey);
   boolean containsKey(K key, HK hashKey);
   CloseableIterable<HK> keys(K key);
}
```

### Factory

`SyncMultimaps` gains typed getters. The collection type in the cache configuration determines which subtype is valid; requesting the wrong type throws at runtime.

```java
interface SyncMultimaps {
   // Returns SET or BAG multimap (default)
   <K, V> SyncMultimap<K, V> get(String name);
   <K, V> SyncMultimapList<K, V> getList(String name);
   <K, V> SyncMultimapSortedSet<K, V> getSortedSet(String name);
   <K, HK, HV> SyncMultimapMap<K, HK, HV> getMap(String name);

   <K, V> SyncMultimap<K, V> create(String name, MultimapConfiguration configuration);
   <K, V> SyncMultimapList<K, V> createList(String name, MultimapConfiguration configuration);
   <K, V> SyncMultimapSortedSet<K, V> createSortedSet(String name, MultimapConfiguration configuration);
   <K, HK, HV> SyncMultimapMap<K, HK, HV> createMap(String name, MultimapConfiguration configuration);

   void remove(String name);
   CloseableIterable<String> names();
   void createTemplate(String name, MultimapConfiguration configuration);
   void removeTemplate(String name);
   CloseableIterable<String> templateNames();
}
```

The same pattern applies to `AsyncMultimaps` with `CompletionStage` and `Flow.Publisher` return types.

### SyncContainer

No changes — `SyncContainer.multimaps()` still returns `SyncMultimaps`, which now has the typed getters.

## 4. Hot Rod Protocol

The wire protocol operations remain generic to avoid proliferation. The collection type is a property of the cache configuration, not of individual commands.

### Existing generic operations (unchanged)

These work for all collection types:

- `MULTIMAP_PUT` — add a value
- `MULTIMAP_GET` — get all values for a key
- `MULTIMAP_REMOVE_KEY` — remove all values for a key
- `MULTIMAP_REMOVE_ENTRY` — remove a specific value from a key
- `MULTIMAP_CONTAINS_KEY`
- `MULTIMAP_CONTAINS_ENTRY`
- `MULTIMAP_SIZE`

### New generic operations

Type-specific access patterns, but with generic wire commands that the server dispatches based on the cache's collection type:

| Operation | Used by | Parameters |
|-----------|---------|------------|
| `MULTIMAP_GET_BY_INDEX` | LIST | key, index |
| `MULTIMAP_PUT_AT_INDEX` | LIST | key, index, value |
| `MULTIMAP_SET_AT_INDEX` | LIST | key, index, value |
| `MULTIMAP_REMOVE_AT_INDEX` | LIST | key, index |
| `MULTIMAP_INDEX_OF` | LIST | key, value |
| `MULTIMAP_SUB_RANGE` | LIST, SORTED_SET | key, from, to |
| `MULTIMAP_SCORE` | SORTED_SET | key, value |
| `MULTIMAP_RANK` | SORTED_SET | key, value |
| `MULTIMAP_GET_BY_INNER_KEY` | MAP | key, hashKey |
| `MULTIMAP_PUT_BY_INNER_KEY` | MAP | key, hashKey, value |
| `MULTIMAP_REMOVE_BY_INNER_KEY` | MAP | key, hashKey |
| `MULTIMAP_INNER_KEYS` | MAP | key |

Calling an operation incompatible with the cache's collection type returns an error.

## 5. Queryable Multimaps

### Problem

Multimap caches store values as `Bucket<V>` where each value is wrapped in `MarshallableUserObject` — opaque bytes that the indexing engine cannot inspect. The query infrastructure already supports indexing and querying repeated/embedded protobuf fields, but the multimap storage format prevents it from being used.

```
Cache<K, Bucket<V>>
         └── repeated MarshallableUserObject wrappedValues  ← opaque, not queryable
```

### Approach

Make the value type transparent to the indexer by generating a type-specific bucket proto schema at cache startup, and expose querying through the multimap API.

### Proto schema (user-authored)

The user declares their value type as a normal `@Indexed` message:

```proto
/**
 * @Indexed
 */
message Tag {
   /**
    * @Keyword
    */
   optional string name = 1;

   /**
    * @Basic
    */
   optional int32 weight = 2;
}
```

### Cache configuration

A multimap cache with indexing enabled, listing the value type as the indexed entity:

```xml
<local-cache name="indexedMultimap">
   <encoding media-type="application/x-protostream"/>
   <indexing enabled="true" storage="filesystem">
      <indexed-entities>
         <indexed-entity>Tag</indexed-entity>
      </indexed-entities>
   </indexing>
</local-cache>
```

Programmatic equivalent:

```java
Configuration config = new ConfigurationBuilder()
      .encoding().mediaType("application/x-protostream")
      .indexing()
         .enable()
         .addIndexedEntity("Tag")
      .build();
```

### Usage

```java
SyncMultimap<String, Tag> tagCache = container.multimaps().create("tags", config);

// Add values
tagCache.add("article-1", new Tag("java", 10));
tagCache.add("article-1", new Tag("infinispan", 8));
tagCache.add("article-2", new Tag("java", 5));

// Query across all keys, filtering by value fields
Query<Object[]> q = tagCache.query(
      "SELECT key, value FROM Tag t WHERE t.name = 'java' AND t.weight > 7");
List<Object[]> results = q.list();
// → [["article-1", Tag("java", 10)]]
```

### Ickle query examples

```sql
-- Find keys that have any value matching a predicate
FROM Tag t WHERE t.name = :name

-- Projection: return the key and matching values
SELECT key, t FROM Tag t WHERE t.weight > 5

-- Aggregation: count entries by tag name across all keys
SELECT t.name, COUNT(t.name) FROM Tag t GROUP BY t.name

-- Combined: keys with at least one high-weight "java" tag
FROM Tag t WHERE t.name = 'java' AND t.weight >= 8
```

The query targets the **value type** (`Tag`), not `Bucket`. The engine iterates across all buckets and matches against individual elements. Each match returns the owning key.

### Internal changes for querying

#### Generated proto schema

At cache startup, when the value entity type is known from `<indexed-entity>`, generate a type-specific bucket schema:

```proto
/**
 * @Indexed
 */
message Bucket_Tag {
   /**
    * @Embedded
    */
   repeated Tag values = 1;
}
```

The `@Embedded` annotation on the repeated field enables the existing query engine to index and search into collection elements — no query parser changes needed.

#### Bucket serialization

When the value type is a known proto message, serialize values directly instead of wrapping in `MarshallableUserObject`. This makes the value fields visible to the indexer.

#### Indexing bridge

Map the generated bucket schema fields into the Hibernate Search index. The repeated `@Embedded` field produces a flattened (or nested, depending on `Structure`) index representation, matching the existing behavior for repeated proto fields.

#### Ickle query semantics

The value type becomes the query root entity. Results carry the owning key. This is analogous to how `FROM Parent p WHERE p.fastChildren.id = 10` already works for embedded repeated fields in regular caches.

## 6. Events and Listeners

### Problem

Cache events carry `CacheEntryEvent<K, Bucket<V>>` — the entire bucket as the value. This is both too much (sending a full list when a single item was added) and too little (no positional or subkey context). Multimap listeners need filtered, type-specific events that describe what changed within the collection, not the whole collection.

### Design

Under the hood, multimap listeners register as internal cache listeners on the underlying `Cache<K, Bucket<V>>`. The multimap event layer intercepts `CacheEntryCreated`/`Modified`/`Removed` events, diffs the old and new bucket, and emits multimap-specific events containing only the delta.

### Event types

```java
enum MultimapEventType {
   ADDED,
   REMOVED,
   MODIFIED
}
```

### Base event

```java
interface MultimapEntryEvent<K, V> {
   K getKey();
   V getValue();
   V getOldValue();
   MultimapEventType getType();
   boolean isBulk();          // true for remove(K) that drops all values
}
```

### Type-specific events

```java
interface MultimapListEvent<K, V> extends MultimapEntryEvent<K, V> {
   long getIndex();
}

interface MultimapSortedSetEvent<K, V> extends MultimapEntryEvent<K, V> {
   double getScore();
   double getOldScore();      // for score updates
}

interface MultimapMapEvent<K, HK, HV> extends MultimapEntryEvent<K, HV> {
   HK getHashKey();
}

interface MultimapArrayEvent<K, V> extends MultimapEntryEvent<K, V> {
   long getIndex();
}
```

### Listener interfaces

```java
@FunctionalInterface
interface MultimapListener<K, V> {
   void onEvent(MultimapEntryEvent<K, V> event);
}
```

Registered on the typed multimap:

```java
SyncMultimapList<String, String> list = container.multimaps().getList("mylist");
list.listen((MultimapListEvent<String, String> event) -> {
   System.out.println(event.getType() + " at index " + event.getIndex() + ": " + event.getValue());
});
```

### Event granularity

- **Single-element operations** (`add`, `remove(K, V)`, `set`, `put-by-inner-key`): one event per affected element, with full positional/subkey context.
- **Index-shifting operations** (e.g. `add(key, 0, value)` on a list): one ADDED event for the inserted element at its index. Shifted elements do not generate events — the logical change is the insertion, not the side effect of shifting.
- **Bulk removal** (`remove(K)` dropping all values): a single event with `isBulk() == true` and `getValue()` returning null. Individual per-value events are not emitted.

### Internals

The event transformation happens on the originating node before the event is dispatched to listeners (including clustered listeners). This ensures only the delta crosses the wire, not the full old+new bucket. The diff logic is collection-type-specific:

| Type | Diff strategy |
|------|---------------|
| SET / BAG | Compare old and new value sets to find additions/removals |
| LIST | Compare old and new lists; identify inserted/removed/replaced indices |
| SORTED_SET | Compare by value identity; detect score changes |
| MAP | Compare by inner key; detect added/removed/modified entries |

## 7. Continuous Queries

### Feasibility

Continuous queries (CQ) on regular caches evaluate a standing Ickle predicate against every cache mutation and emit join/leave/update events. Extending this to multimaps is possible but introduces complications that vary by collection type.

### Challenges

**Matching granularity.** A regular CQ matches against entire cache entries (`<K, V>`). A multimap CQ must match against individual elements within a bucket. Adding one value to a key can cause that key to start matching — but the key already existed in the cache. The CQ must track per-key, per-element match state: a two-level map instead of a simple set of matching keys.

**Join/leave semantics for bags.** Consider `FROM Tag t WHERE t.name = 'java'` on a bag with `[java, java]`. Removing one `java` still matches — the key doesn't leave the result set. The CQ must track match cardinality for bags, not just presence, to correctly determine when a key truly leaves.

**Index-shifting in lists.** Inserting at index 0 shifts all subsequent indices. A CQ that projects indices would need to decide whether to re-emit events for every shifted element. This can be extremely chatty and surprising to consumers.

**Score changes in sorted sets.** Changing an element's score can cause it to enter or leave a range-based CQ (`WHERE score > 5`). This is well-defined but requires comparing old and new scores on every mutation.

**Diff cost.** Every `put`/`remove` triggers a bucket-level cache event. The CQ engine must diff old vs. new bucket on every mutation to determine element-level joins and leaves. For large buckets on hot keys, this can be expensive.

### Recommended scope

Support continuous queries for **SET**, **SORTED_SET**, and **MAP** initially. These types have stable element identity (value for sets, value+score for sorted sets, inner key for maps), which makes join/leave semantics clean and unambiguous.

Defer **BAG** (duplicate cardinality tracking is complex and the semantics are hard to explain to users) and **LIST** (index-shifting makes join/leave noisy and potentially misleading). These can be added later if there's demand, with clearly documented behavior for edge cases.

### API

```java
SyncMultimap<String, Tag> tagSet = container.multimaps().get("tags");
ContinuousQuery<String, Tag> cq = tagSet.continuousQuery();
cq.addListener("FROM Tag t WHERE t.weight > 5", new MultimapContinuousQueryListener<>() {
   @Override
   public void onJoin(K key, V value) { /* element now matches */ }

   @Override
   public void onLeave(K key, V value) { /* element no longer matches */ }

   @Override
   public void onUpdate(K key, V value) { /* element still matches but changed */ }
});
```

## 8. Cross-cutting Concerns

### Cross-site replication and conflict resolution

When two sites concurrently modify the same multimap key, the standard `EntryMergePolicy` operates on the whole bucket value. Collection-type-specific merge strategies are needed:

| Type | Merge feasibility | Strategy |
|------|-------------------|----------|
| SET | Straightforward | Union, intersection, or site-priority — all well-defined on sets |
| BAG | Moderate | Union is the natural default, but cardinality conflicts are ambiguous (site A adds 2, site B adds 3 — result has 2, 3, or 5?) |
| LIST | Difficult | Concurrent insertions at different indices have no natural merge order. Append-only (concatenate) is safe; arbitrary index operations are not |
| SORTED_SET | Straightforward | Merge by value identity; for conflicting scores, take highest, lowest, or site-priority |
| MAP | Moderate | Merge by inner key; conflicting values for the same inner key need a sub-merge policy |

Recommendation: provide built-in merge strategies for SET and SORTED_SET. For LIST, only support append-mode merging and reject index-based operations in cross-site configurations (or document them as last-write-wins on the whole bucket). For MAP, delegate inner-key conflicts to a user-supplied function or default to site-priority.

### Collection size limits

Without limits, a single key's bucket can grow unboundedly, causing memory pressure and large serialization payloads. A configurable `max-size` per collection should be supported, with a type-specific overflow policy:

```xml
<distributed-cache name="bounded-list">
   <multimap:list max-size="10000" overflow="REJECT"/>
</distributed-cache>
```

| Overflow policy | Behavior |
|-----------------|----------|
| `REJECT` | Throw an exception when the limit is reached |
| `EVICT_OLDEST` | Remove the oldest element (LIST head, or oldest insertion for SET/BAG) |
| `EVICT_LOWEST` | Remove the lowest-score element (SORTED_SET only) |

This is the one case where an attribute on the multimap element is justified — `max-size` and `overflow` are numeric/enum properties of the collection, not variant selectors.

### Persistence and write amplification

Buckets are stored as single entries in cache stores. Every single-element `add` or `remove` rewrites the entire serialized bucket. For large collections this causes:

- **Write amplification**: adding one element to a 100K-element list rewrites all 100K elements to the store.
- **Store size limits**: some backends (JDBC with VARCHAR columns, S3 with object size limits) may reject very large blobs.
- **State transfer**: large buckets slow down rebalancing when nodes join or leave.

Possible mitigations:
- **Delta-based persistence**: store element-level operations as a log, periodically compacting to a full snapshot. This is a significant architectural change.
- **Chunked storage**: split large buckets across multiple store entries with a manifest. Adds complexity to the store SPI.
- **Documentation**: for the initial implementation, document that multimap caches are best suited for moderate collection sizes (thousands, not millions) and that very large collections should use a regular cache with application-managed sharding.

The pragmatic first step is to rely on `max-size` limits and document the tradeoff. Delta persistence or chunking can be explored later if demand warrants it.

### Expiration

Lifespan and max-idle expiration apply to the whole bucket — all elements for a key share the same expiration. Per-element expiration (e.g., individual TTLs on sorted set members) would require tracking expiration metadata per element within the bucket and integrating with the expiration manager at a sub-entry level.

Recommendation: defer per-element expiration. Whole-bucket expiration covers the common case. Document that if per-element TTLs are needed, users should model each element as a separate cache entry with a compound key.

### Transcoding

The existing TODO (ISPN-11452) notes that multimaps don't support transcoding. The queryable multimap design requires protostream encoding, but users accessing via REST or Hot Rod may want JSON or other formats. With typed buckets and known value types, the server can transcode individual values between formats on read/write — the value type schema provides the necessary metadata. This should be addressed as part of the transparent bucket serialization work.

### REST API

Type-specific operations need REST endpoints. Proposed URL patterns:

| Operation | Method | URL |
|-----------|--------|-----|
| Add value | `POST` | `/v2/caches/{cache}/multimap/{key}` |
| Get all values | `GET` | `/v2/caches/{cache}/multimap/{key}` |
| Remove key | `DELETE` | `/v2/caches/{cache}/multimap/{key}` |
| Get by index (LIST) | `GET` | `/v2/caches/{cache}/multimap/{key}?index={n}` |
| Get by inner key (MAP) | `GET` | `/v2/caches/{cache}/multimap/{key}/{hashKey}` |
| Get by score range (SORTED_SET) | `GET` | `/v2/caches/{cache}/multimap/{key}?minScore={s}&maxScore={s}` |
| Query | `GET/POST` | `/v2/caches/{cache}/multimap?action=search` |

### Remote client parity

The existing `RemoteMultimapCache` needs the same typed extensions as the embedded API (`RemoteMultimapList`, `RemoteMultimapSortedSet`, `RemoteMultimapMap`), backed by the new Hot Rod operations. The remote client should mirror the `api` module interfaces so that embedded and remote usage are symmetrical.

### Query requires indexing

Querying multimap values is supported **only** when indexing is enabled on the cache. Non-indexed multimaps can still be used for all CRUD operations but do not support Ickle queries or continuous queries. This is consistent with how the query engine works for regular caches — the transparent bucket serialization and generated proto schema are only activated when `<indexing enabled="true">` is configured.

### Near-caching

Hot Rod near-caching operates at the cache entry level — it caches `<K, Bucket<V>>` on the client. For multimaps this means:

- A single-element add/remove invalidates the entire near-cached bucket for that key, even though only one element changed.
- For large collections on frequently accessed keys, this causes excessive invalidation traffic and re-fetching of the full bucket.

Possible mitigations:
- **Element-level near-cache deltas**: the server sends only the element-level change (add/remove/modify) and the client patches its local bucket. This requires protocol support for delta notifications, which doesn't exist today.
- **Whole-bucket invalidation (initial approach)**: accept the overhead for the initial implementation and document that near-caching is most effective for multimaps with small to moderate collection sizes.

### Web Console

Multimaps are caches under the hood, but the console should present them differently — the data view needs to show key → collection structure rather than key → single value. The collection type determines the appropriate visualization (list with indices, sorted set with scores, map with inner keys, etc.). Exact UX is TBD and should involve the UX team.

### Security and authorization

No special handling needed. Multimap operations are cache operations, so the existing RBAC roles and permissions (READ, WRITE, BULK_READ, etc.) apply as-is. The new type-specific operations (index-based access, inner-key access, etc.) map to the same permission categories as their generic cache equivalents.

### CLI

No special handling needed. Multimaps are caches with extra configuration, so the existing `create cache`, `ls caches`, `describe cache/name` commands work as-is. The multimap collection type is visible in the cache configuration output.

### Quarkus integration

Evaluate whether a dedicated `@Multimap` CDI annotation is needed, or whether the existing cache injection mechanism is sufficient since multimaps are caches with extra configuration. If a dedicated annotation is added, it should support the typed variants (e.g., injecting `SyncMultimapList<K, V>` directly).

### Deprecations

The following classes are deprecated in favor of the new `api` module interfaces:

- `org.infinispan.client.hotrod.multimap.RemoteMultimapCache`
- `org.infinispan.client.hotrod.multimap.RemoteMultimapCacheManager`
- `org.infinispan.client.hotrod.multimap.RemoteMultimapCacheManagerFactory`
- `org.infinispan.client.hotrod.multimap.MultimapCacheManager`
- `org.infinispan.client.hotrod.multimap.MetadataCollection`

The embedded `MultimapCache` and `MultimapCacheManager` in the `multimap` module remain as-is for backward compatibility but should also be deprecated once the new API is stable.

### Rolling upgrades and migration

Existing multimap caches use `MarshallableUserObject`-wrapped buckets. Upgrading to typed buckets requires a migration path:

- The new bucket format should be able to read old `MarshallableUserObject`-wrapped data and lazily convert on first access.
- During rolling upgrades, the source cluster serves old-format buckets; the target cluster converts on ingest.
- A cache store migrator step may be needed for offline migration of persistent stores.

## 9. Summary of changes by layer

| Layer | Change |
|-------|--------|
| XML schema | Add `multimap` namespace with closed set of elements: `set`, `bag`, `list`, `sorted-set`, `map` (and `array` in the future) |
| `api` module | Replace single `SyncMultimap`/`AsyncMultimap` with base + typed extensions (List, SortedSet, Map) |
| `api` configuration | Add `CollectionType` enum to `MultimapConfiguration` |
| `api` factory | Add typed getters/creators to `SyncMultimaps`/`AsyncMultimaps` |
| `multimap` module | `MultimapCache` API unchanged; internal bucket types already exist |
| Hot Rod protocol | Add generic operations for index-based, score-based, and inner-key access |
| Proto schema | Generate `Bucket_<ValueType>` with `@Embedded repeated <ValueType> values` for indexed multimaps |
| Bucket serialization | Serialize values directly when value type is a known proto message |
| Indexing bridge | Map generated bucket schema fields into the Hibernate Search index |
| Ickle semantics | Value type becomes the query root entity; results carry the owning key |
| Events | Type-specific multimap events with delta-only payloads, built on top of cache entry events |
| Continuous queries | Element-level join/leave/update over standing Ickle predicates; SET, SORTED_SET, and MAP initially |
| Near-caching | Whole-bucket invalidation initially; element-level deltas deferred |
| Web Console | Collection-aware data view for multimaps; UX TBD |
| Quarkus | Evaluate `@Multimap` CDI annotation or reuse existing cache injection |
| Deprecations | Deprecate old `RemoteMultimapCache*` and `MultimapCacheManager` classes |
| Query parser | No changes — repeated/embedded field querying already works |
