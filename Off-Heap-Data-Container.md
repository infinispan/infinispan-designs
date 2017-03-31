Currently the data container that holds all of the entries in memory stores them in the Java heap.  This area is subject to garbage collection and others.  Unfortunately the default garbage collector can cause stop the world pauses where it can halt the entire JVM from processing.

The easiest way to avoid these performance issues is to not store these entries in the Java Heap.  Easier said than done!  Below you will find a few ways that you could do this

1. Use a third party tool to implement DataContainer (ie. MapDB, ~~ChronicleMap~~ etc.) - http://stackoverflow.com/questions/7705895/is-there-a-open-source-off-heap-cache-solution-for-java
2. Allow for data container to store nothing and only ever go to Store.  Then we could utilize something like SIFS using a filesystem pointing to ramcache
3. Implement our own using netty as a an example of how to do the memory allocations.

# Things that are incompatible

### Write skew check
Currently this relies on object instance equality
### Atomic Map
This updates instance directly, thus ICE factory update has to make a new instance.  Also proxy requires some additional changes.  This doesn't seem to work properly also because of Read Committed below...
### Class Resolver
Can't figure out quite yet how to get the class loader to work properly with new DBContainer
### Read Committed
Read committed cannot be used since everything is serialized when stored.  Only repeatable read may be used.  This is besides the fact that read committed has issues in a DIST cache anyways.


## Map DB array perf
Result: 251528.669 ±(99.9%) 3849.842 ops/s [Average]
  Statistics: (min, avg, max) = (204159.349, 251528.669, 271807.636), stdev = 16300.471
  Confidence interval (99.9%): [247678.827, 255378.510]


Run complete. Total time: 00:13:36

Benchmark|                      (cacheName)|   Mode|  Cnt|        Score|       Error|  Units|
---------|---------------------------------|--------|----|--------------|-----------|--------|--
IspnBenchmark.testGet|           nonTxCache|  thrpt|  200|  1657296.817| ± 26651.726|  ops/s|
IspnBenchmark.testPutImplicit|   nonTxCache|  thrpt|  200|   251528.669| ±  3849.842|  ops/s|

## MapDB Direct perf

Result: 106203.466 ±(99.9%) 1703.474 ops/s [Average]
  Statistics: (min, avg, max) = (86628.725, 106203.466, 117797.812), stdev = 7212.614
  Confidence interval (99.9%): [104499.992, 107906.939]


Run complete. Total time: 00:13:37

Benchmark|                      (cacheName)|   Mode|  Cnt|        Score|       Error|  Units|
---------|---------------------------------|--------|----|--------------|-----------|--------|--
IspnBenchmark.testGet|           nonTxCache|  thrpt|  200|  1608557.747| ± 21212.189|  ops/s|
IspnBenchmark.testPutImplicit|   nonTxCache|  thrpt|  200|   106203.466| ±  1703.474|  ops/s|


## Master perf
Result: 1318669.580 ±(99.9%) 17467.592 ops/s [Average]
  Statistics: (min, avg, max) = (1043144.741, 1318669.580, 1453850.008), stdev = 73958.877
  Confidence interval (99.9%): [1301201.989, 1336137.172]


Run complete. Total time: 00:13:35

Benchmark|                      (cacheName)|   Mode|  Cnt|        Score|       Error|  Units|
---------|---------------------------------|--------|----|--------------|-----------|--------|--
|IspnBenchmark.testGet|           nonTxCache|  thrpt|  200|  20555714.561| ± 274213.492|  ops/s|
IspnBenchmark.testPutImplicit|   nonTxCache|  thrpt|  200|   1318669.580| ±  17467.592|  ops/s|

## SFS perf
Result: 74038.493 ±(99.9%) 1512.844 ops/s [Average]
  Statistics: (min, avg, max) = (42949.523, 74038.493, 81948.143), stdev = 6405.476
  Confidence interval (99.9%): [72525.649, 75551.337]


Run complete. Total time: 00:13:36

Benchmark|                      (cacheName)|   Mode|  Cnt|         Score|        Error|  Units|
---------|---------------------------------|--------|----|--------------|-----------|--------|--
IspnBenchmark.testGet|           nonTxCache|  thrpt|  200|  774529.207| ± 12164.148|  ops/s|
IspnBenchmark.testPutImplicit|   nonTxCache|  thrpt|  200|   74038.493| ±  1512.844|  ops/s|

