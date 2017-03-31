Infinispan currently only stores entries on the Java heap.  This can cause issues if the cache is used in such a way that a large amount of stop the world garbage collections are performed.  One such fix is to write these objects to a cache store, however this can cause performance issues.  Instead Infinispan would like to instead of storing entries on the Java heap but rather to store them in off heap memory managed by Infinispan itself.  This would allow for more memory to be used and should reduce significantly the number of garbage collections required.


This requires quite of a bit of work to be done in the core module of Infinispan.  Before I go into the details of how the overall design for core will go I will list some limitations

## Limitations

1. Write skew check
Currently this relies on object instance equality, which will not work since objects are required to be deserialized/serialized
2. Atomic Map
This updates instance directly, thus ICE factory update has to make a new instance.  Also proxy requires some additional changes.  This doesn't seem to work properly also because of Read Committed below...
3. Read Committed
Read committed cannot be used since everything is serialized when stored.  Only repeatable read may be used.  This is besides the fact that read committed has issues in a DIST cache anyways.
4. DeltaAware
DeltaAware would technically work, but it will take a significant performance hit due to the fact that objects would have to be fully deserialized then apply delta and then serialized again.
5. Store as Binary
Both this and off heap together do not make sense

## Configuration

Off heap memory would be configured under a new element ("STORAGE"?, "MEMORY"?, under data container?) along with storeAsBinary and wrapByteArray.  The wrapByteArray is a new configuration to replace equivalence that allows for byte[] to be used as key or values.  This element is mutually exclusive as it wouldn't make sense to use them together.  Compatibility would fall under this but we are planning on removing Compatibility in Infinispan 9 as well

## Put operation

The general idea for core is as following for a general put:

1. Originator immediately deserializes the object as a wrapped byte[] (Do we use hashCode for byte[] or object?)
2. Originator sends wrapped byte[] to the primary owner and normal forwarding occurs
3. The primary and backup owners have an interceptor installed that converrts the byte[] to an off heap object before writing this instance to the data container
4. The previous value (if needed) is sent back as a byte[] and the originator finally deserializes it before returning it to the originator

## Get operation

A get will work in a similar fashion:

1. Originator immediately deserializes the object as a wrapped byte[] (Do we use hashCode for byte[] or object?)
2. Originator sends wrapped byte[] to the owner(s) to retrieve the value
3. Hacky but the wrapped byte[] can compare equivalence to the off heap object without having to allocate a byte\[\] (indexed gets) - (TODO: check bulk get byte[] - requires read only copy though)
4. Originator deserializes byte[] to an object before returning to caller

## Bulk operations

Bulk methods such as distributed streams will require deserialization of entries on each node.  If we store the hash along with the off heap instance we do not need to deserialize non owned keys as the hash code would determine which segment it is in.

## Statistics

We need to make statistics available that show how much direct memory is in use (blocks? available? used?).  Unfortunately this may have an issue with separating information by cache level and may only provide per node usage.

# Server considerations

The server doesn't require as many changes as in the core, but there are a few things to implement and investigate

## Reuse Netty buffer

Netty can utilize direct byte buffers for its data read from the socket.  This can be useful to reuse these to prevent an additional byte[] allocation.  It is unclear if this will work as intended, but if possible this will be added.  Even if this works it may not save any performance however for writes as JGroups currently only allows for byte[] to be written and not off heap memory.

## Compatibility

Compatibility cannot be enabled with off heap and will throw an exception preventing the cache from starting.