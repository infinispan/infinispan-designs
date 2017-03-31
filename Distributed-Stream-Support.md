# Intro
As of Infinispan 7, we have support for distributed entry retrieval using the EntryRetriever framework.  This retriever allows for distributed computation for filtering and conversion of entries.  This works well enough when the only desire is to a 1:1 representation of key and values.  However it may be more beneficial instead to perform more complex operations and allow for results that do not map 1:1 to the number of keys.

# Proposal
Infinispan 8 would add a new distributed based [stream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html) implementation.  Collections retrieved using keySet, entrySet and values would all support this.  

Currently Infinispan returns a CloseableIteratorSet, CloseableIteratorCollection for the previously mentioned methods.  A new CacheSet or CacheCollection (names to be decided) would be returned instead.  These new collections would return a CacheStream (names to be decided) when either the stream or parallelStream method is invoked.

This is a possible definition of the CacheStream interface.

```java
interface CacheStream<R> extends Stream<R> {
    // This would disable sending multiple requests at the same time, which would reduce memory usage on the 
    // originator node as it has to keep intermediate data
    CacheStream<R> sequentialDistribution();
    // Will only run the stream on the given keys.  This is done before the stream is distributed or any cache
    // loader is used to reduce overhead as needed
    CacheStream<R> filterKeys(Set<?> keys);
    // Only entries that map to given segments are operated upon.  Similarly to filterKeys this is done 
    // before the stream is processed to reduce overhead
    CacheStream<R> filterKeySegments(Set<Integer> segments);
    // Controls how many keys are returns from a remote node at at time.  Note this option is ignored for
    // terminal operators that don't deal with 1:1 mappings with keys (such as count, reduce, collect etc.)
    CacheStream<R> distributedBatchSize(long batchSize);
    // Allows registration of a segment completion listener that is notified when a segment has completed
    // processing.  If the terminal operator has a short circuit this listener may never be called.
    CacheStream<R> segmentCompletionListener(SegmentCompletionListener listener);
    // When this is enabled keys or segments are not tracked and if a rehash occurs there may be some 
    // data that is not processed or possibly duplicated.  This will be faster in almost all cases but however
    // doesn't guarantee correctness under node failure.
    CacheStream<R> disableRehashAware();
}
```

The filterKeySegments and filterKeys methods are done before sending anything remote to verify that the proper segments are requested and only keys from those owners are requested.  These methods are also utilized as a KeyFilter for the cache loader filter to prevent reading other keys if the loader supports a given filter.

This new CacheStream would allow for all Stream methods except distinct and sorted are not supported (distinct likely will never be supported and sorted will be added later).  The usefulness of the limit and skip intermediate operations are also dubious, since until sorted is implemented it would be non deterministic which entries would be returned.  All operations would be normally performed on the node that owns the data.

## General Algorithm for Rehash Aware Stream
Running the stream on each node independently is not that interesting.  The interesting part is how we track entries as they move between nodes due to the addition or removal of a new node.

### Originator steps:

1. Determines what segments it must request from both the requested segments and keys (if neither is provided then all keys are considered)
2. Depending on if parallel distribution is enabled it will either start local stream operations and if enabled will send a request to each node for the segments it owns.  If parallel distribution is disabled then the originator will send each request after each node has finished processing its data set.
3. If a node was found to be suspect this node will keep track of each key it has processed for the given segment and subsequently send a request to the new owner asking for it to perform its operation again without those keys.  Nodes that have partially retrieved segments may be prioritized first.

### Remote node steps:

1. We obtain a Stream from the cache (Flag.LOCAL_MODE forced).
2. We apply the given segment and/or key filter first.  Note that these are used on the originator to determine what nodes to request data from and then subsequently sent to target node to reduce data at the first possible step
3. Assuming that fault tolerance is enabled and we are using a terminal operator that requires key knowledge (ie. convert, reduce), we keep track of a given key's value or segment.
4. All subsequent user provided stream operations are performed.
5. As values are found from the stream we send them back as they reach the buffer size (key aware terminal operators only).  See the below table for terminal operators that do not do this.
6. If a rehash occurs in the middle that given segment is considered suspect and on the next response to the originator will be told of it being so.  The node that owned that segment may or may not send back values for keys that map to the given segment.

### This table is a brief description of what each intermediate operation would do

| Method | Behavior |
|--------- | ------------ |
| distinct | This operation would be extremely costly in a distributed environment as all data up to that point of the stream must be read by a single node.  Due to the memory and performance constraints this method wouldn't be supported. |
| filter | This is ran on each data owner node |
| flatMap | This is ran on each data owner node.  **EXCEPTION** This intermediate operator changes the behavior of the provided terminal operator to be Segment based tracking only.  This is a limitation due to the nature of returning multiple values possibly, making key tracking much more difficult.  This may be resolved by doing some multiple stream composition using ForkJoinPool, however this may come later. |
| limit | This method is more useful when using an ORDERED stream, which will be added when sorted is supported. |
| map | This is ran on each data owner node |
| peek | Note this method is ran in the distributed environment, which may not be what is normally desired. |
| skip | This method is more useful when using an ORDERED stream, which will be added when sorted is supported.  See https://issues.jboss.org/browse/ISPN-4358 |
| sorted | This is not supported, yet!  See  |

### This table is a brief description of what each terminal operation would do

| Method | Tracking Type | Behavior |
|--------- | ------------ | ------------ |
| allMatch | Segment | Each node keep track of given segments.  If while iterating a value is not found to match, false is immediately returned.  However if all pass at the end of iteration a response is sent stating that all segments that this node kept during the entire iteration passed.  Any suspected nodes are told to the originator which then must resubmit the operation for those segments to run again. |
| anyMatch | Segment | If a node finds a value that matches it would immediately return that value.  If no values match than the node will send a response back to the owner stating which segments had no match. |
| collect | Segment | The combiner is ran until completion on each node.  If a rehash occurs in the middle the current results are abandoned.  A response is sent back to the originator that those segments were suspect and the stream operation(s) begins again.  A response is not generated until a topology is in place for the entire iteration. |
| count | Segment | The count is ran until completion on each node.  If a rehash occurs in the middle the current results are abandoned.  A response is sent back to the originator that those segments were suspect and the stream operation(s) begins again.  A response is not generated until a topology is in place for the entire iteration. |
| findAny | Segment |Returns the first value that is returned.  If no value is found then those segments are marked as completed.  If a rehash occurs only the segments that were valid during the operation are completed and others are suspected. |
| findFirst | Segment | Same as findAny |
| forEach | Key | This operation does full tracking of keys and reports which keys have been processed intermittently controlled by the distributedBatchSize.  This operation guarantees that each value is ran at least once, unfortunately if a rehash occurs the operation could be ran more than 1 time per value. |
| forEachOrdered | | This operation is not yet supported, forEach should be used. |
| iterator | Key | This operation does full tracking of keys and reports which keys have been processed intermittently controlled by the distributedBatchSize.  This operation guarantees that each value is ran at least once, unfortunately if a rehash occurs the operation could be ran more than 1 time per value.  Thus the originator will internally keep track of each key by segment to guarantee that a given key is not seen provided to the iterator more than once. |
| max | Segment | This would keep track of the maximum entry by segment.  When segment(s) is completed it would send the highest entry for the given segments to the originator.  The originator would then find the highest entry to return. |
| min | Segment | Same as max, but would find the minimal entry. |
| noneMatch | Segment | Same as all match, but inverted |
| reduce | Segment | The accumulator is ran until completion on each node.  If a rehash occurs in the middle the current results are abandoned.  A response is sent back to the originator that those segments were suspect and the stream operation(s) begins again.  A response is not generated until a topology is in place for the entire iteration.  The combiner is ran finally on the originator node for all of the results. |
| spliterator | Key | This initially would just use the iterator, however future improvements could be done to make this partition data by segment or node in the future. |
| toArray | Key | This would internally use a spliterator or iterator to retrieve the values and then finally process them into an array.  This terminal operator is only invoked locally. |

