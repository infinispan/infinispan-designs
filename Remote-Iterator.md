## Intro
(by Tristan)

Currently the HotRod protocol implements a few bulk operations (BulkGetRequest, BulkGetKeysRequest, QueryRequest) which return arrays of data. The problem with these operations is that they are very inefficient since they build the entire payload on the server, thus potentially requiring a lot of transient memory, and then send it to the client in a single response, which again might be to large to handle. It is also not possible to pre-filter the data on the server according to some criteria to avoid sending to the client unneeded entries.

For symmetry with the embedded distributed iterator, HotRod should implement an equivalent remote distributed iterator which would:

- allow a client to specify a filter/scope (local, global, segment, custom, etc)
- allow the client and the server to iterate through the retrieved entries using an iterator paradigm appropriate to the environment (java.lang.Iterator in Java, <iterator> in C++, IEnumerator in C#) in a memory efficient fashion
- allow a server to efficiently batch entry in multiple responses
- leverage the already existing KeyValueFilter and Converters, including deployment of custom ones into the server

## Proposal

### API changes

A new method on `org.infinispan.client.hotrod.RemoteCache`:

```java
/**
  * Retrieve entries from the server
  *
  * @param filterConverterFactory Factory name for {@link KeyValueFilterConverter} or null
  * @param segments The segments to iterate on or null if all segments should be iterated
  * @param batchSize The number of entries transferred from the server at a time
  */
CloseableIterator retrieveEntries(String filterConverterFactory, Set<Integer> segments, int batchSize);
```
<br>
### HotRod Protocol

##### Operation: ITERATION_START

Request:

| Field name      | Type    | Value |
| ----------------| --------| ------
| SEGMENTS        | byte[ ] | segments requested 
| FILTER_CONVERT  |  String | Factory name for FilterConverter
| BATCH_SIZE      | vInt    | batch to transfer from the server

>About segments

> One way to encode the list of segments ids to be sent is to use 
> N bits to represent each of the N segments, being a flipped bit a desired segment id. With the default number of segments(60) it'd use only 60 bits, but if the number of segments is gigantic this could cause issues.

Response:

| Field name      | Type    | Value |
| ----------------| --------| ------
| ITERATION_ID    | String  | UUID of the iteration

<br>

##### Operation: ITERATION_NEXT

Request:

| Field name      | Type    | Value |
| ----------------| --------| ------
| ITERATION_ID    | String  | UUID of the iteration 

Response:

| Field name      | Type    | Value |
| ----------------| --------| ------
| ITERATION_ID      | String  | id of the iteration
| FINISHED_SEGMENTS | byte[ ]  | segments that finished iteration
| ENTRIES_SIZE      | vInt    | number of entries transfered
| KEY 1               | byte    | entry 1 key
| VALUE 1             | byte    | entry 1 value
|  ...               |  ...    |  ...
| KEY N               | byte    | entry N key
| VALUE N             | byte    | entry N value   

<br>
##### Operation: ITERATION_END

Request:

| Field name      | Type    | Value |
| ----------------| --------| ------
| ITERATION_ID    | String  | UUID of the iteration 

Response:

| Field name      | Type    | Value |
| ----------------| --------| ------
| STATUS    | byte  | ACK of the operation 

<br>
### Client side 

Upon calling ```retrieveEntries``` an ITERATION_START op is issued and the client will store the tuple ```[Address, IterationId]``` so that all subsequent operations will go to the same server.

```CloseableIterator.next()``` will receive a batch of entries and will keep the keys internally on a per segments basis. 
Whenever a segments is finished iterating (received on field FINISHED_SEGMENTS of ITERATION_NEXT response), those keys will be discarded. 
<br>
### Server Side
A ITERATION_START request will create and obtain a ```CloseableIterator``` from a ```EntryRetriever``` with the optional ```KeyValueFilterConverter```, batch size, and set of segments required. The ```CloseableIterator``` will be associated with a UUID and will be disposed upon receiving a ITERATION_CLOSE request. The ```CloseableIterator``` will also have a ```SegmentListener``` associated so that it is notified of finished segments and send them back the client on the ITERATION_NEXT responses.

An ITERATION_CLOSE will dispose the ```CloseableIterator```

### Failover
If the server backing a ```CloseableIterator``` dies, the client will restart the iteration in other node, filtering out already finished segments.

One drawback of the per-segment iteration control is that if a particular segment was not entirely iterated before the server goes down, it will have to be iterated again, leading to duplicate values.

One way to avoid duplicates values to the caller of ```CloseableIterator.next()``` is to have the client to store the keys of the unfinished segments and use them to filter out duplicates after the failover happens.

### References

JIRA: [ISPN-5219] (https://issues.jboss.org/browse/ISPN-5219)

