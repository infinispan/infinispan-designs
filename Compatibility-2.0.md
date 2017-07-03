The current compatibility mode is implemented as follows:
- it adds a TypeConverterInterceptor to the interceptors which perform automatic type conversion on the data in the cache through TypeConverter classes
- TypeConverters are "service loaded", although the interceptor only supports one for each of three supported "modes" (hotrod, memcached, embedded)
- which TypeConerter is used is chosen based on the presence of protocol-specific flags (OPERATION_HOTROD and OPERATION_MEMCACHED) on the commands
- it assumes that the conversion of objects is performed via a Marshaller

The above approach has the following limitations:
- when using Hot Rod, it unmarshalls the incoming byte[] to a Java object before storing it into the cache. This means that all subsequent retrievals of that value over Hot Rod will require converting back to byte[] before being sent over the wire. Since this is going to be the most common type of operation, it is quite inefficient.
- since cache entries don't have any type information metadata, it delegates type detection (if any) to the marshaller
- it cannot provide on-demand data reconversion for clients/protocols

Internal users of type information/conversions would be:
- indexing/query
- scripting

The "fix" is to move to explicit conversion, so that the various components/internals are aware of the original encoding of the data and apply any required conversions on the fly. This requires type information to be added to the cache and/or entries. For space efficiency, the ideal situation is for all entries in a cache to be of homogeneous type, which should either be specified upfront in configuration, or determined implicitly by the first write operation. Support for heterogeneous per-entry type information would mean storing this information in the entry metadata (hopefully using an indexed registry to minimize storage). 

An example of storing type information and performing on-demand type conversion is the REST server: since PUT/POST requests have a Content-Type header, this information is stored in a dedicated Metadata class. Additionally GET requests can specify an optional Accept header which determines the desired type. The server can perform a number of conversions to common types (JSON, XML, text/plain).

The protocol servers will not override the internal Cache representation but use a decorated "transcoding" Cache which uses a transcoder to encode the Cache data in the format (mime-type) requested by the client.
Clients might request operations (with a custom mime-type) to return data using a different Transcoder.
When it’s possible, we’ll automatically pick the "right" transcoder by matching requestor’s mime-type with the known (static) encoding of the Cache.
Some transcoder choices might require additional configuration to be made available as valid choices. E.g. protobuf will need a schema to be deployed, when missing that would be an illegal request.
Embedded Mode also might use a decorated Cache using a Transcoder. For example the Cache might be set to encode data according to a protobuf schema (provided such a schema is given by the user), yet an embedded mode  client might request data in POJO format.

We must ensure that Consistent-Hash-aware clients knowledge of the hashing of keys is not affected by transcoding.

Transcoders should be deployable


# Encoders, Transcoders and indexable Json 

[ISPN-7896](https://issues.jboss.org/browse/ISPN-7896) introduced the Encoders, adding capability to the caches for on-demand data conversion instead of relying on implicit conversions based on a combination of flags and interceptors. 

This section will expand on the Encoders usage and will introduce the complementary Transcoders, used to convert between different encodings.

## About Encoders

An Encoder is essentially a two-way conversion between the storage format in a cache and the read/write format. The ```AdvancedCache```  interface has a method to specify an encoder to be used during all operations involving that cache:

```cache.getAdvancedCache().withEncoding(MyEncoder.class)```

which returns a decorated cache instance where all operations (read/write/stream/add listener, etc) flow through the supplied encoder.

At the moment, there is no cache configuration to specify an Encoder; Infinispan tries to pick the best decorated cache depending on the
situation where this cache is used. For example:

* Embedded caches with ```Storage.OBJECT``` are associated with the ```IdentityEncoder```, which stores objects "as is";
* Embedded caches with ```Storage.OFF_HEAP``` are associated with a ```MarshallerEncoder```, which marshalls data before storing in order to produce a ```byte[]``` (hard requirement);
* Embedded caches with ```Storage.BINARY``` are associated with the BinaryEncoder, which produces a mix of marshalled and unmarshalled objects.
* Caches used in the server with indexing enabled and compatibility mode enabled uses a ```ProtostreamCompatEncoder```, where the byte[] arriving from the client
 is unmarshalled via a protostream marshaller into a java representation.
* Caches used in the server with no indexing and compatibility use the ```IdentityEncoder```, which treats content read and written as opaque ```byte[]```.
* Caches used in the server with compatibility (and no indexing) uses the CompatModeEncoder, which takes the ```byte[]``` from the client and unmarshalls them to java object for storage.
* Scripts which carry data-type ```utf-8``` wraps a cache with the ```UTF8Encoder```, that will extract ```byte[]``` out of a String in order to store and will create Strings out of ```bytes[]``` for reading
operations.


Some endpoints in the Infinispan Server allow to send and request data in a certain format (MimeType):

* Scripting: scripts (.js at least) can pass a header "dataType" with a MimeType string;
* Rest: request headers "content-type" (for PUT/POST) and "accept" (for GET) can carry mime types as well
* HotRod: there is no such capability yet, but it's something that will eventually be added.


How does the Mimetype relate to the encoder general mechanism?


## Proposal

The proposal is to use a pair (Encoder, MimeType) to describe cache storage. Examples: 

* "```application/json``` with no encoding" means IdentityEncoder associated of a content that represents a json (```byte[]```, ```Gson```, ```Map<String, Map<?,?>>```, etc)
* "```text/plain; charset=utf-8``` with marshalling" means content produced by the MarshallerEncoder that represents text, so it's a ```byte[]```
* "```application/octet-stream``` with KryoEncoder" means ```byte[]``` produced by a KryoMarshaller.

This means that different MimeTypes can be associated with the same Encoder, e.g. the IdentityEncoder can be associated with any MimeType meaning whatever is sent
to cache will be written or read without transformations.

The purpose of the MimeType is twofold:

* Provide a metadata about the content of the cache, independently of the encoding;
* By describing the format, enable the conversion between different encodings, a.k.a. Transcoding.


### Configuration changes

A cache will have a data-type section that looks like the following:

```xml
<local-cache>
  <data-type>
     <key mime-type="application/xml" encoder="org.infinispan.commons.dataconversion.IdentityEncoder"/>
     <value mime-type="text/plain; charset=UTF-8" encoder="org.infinispan.dataconversion.MarshalledEncoder"/>
  </data-type>
</local-cache>
```

The MimeType will be either present in the config or inferred; if not provided, Infinispan will automatically pick the best 
decorated cache depending on the situation.


### API changes
Apart from being able to use a different encoders on the fly, it will be possible to ask for a different MimeType using the method:

```java
// Both keys and types as "application/xml"
cache.getAdvancedCache.withMimeType("application/xml").withEncoding(IdentityEncoder.class)

// Different MimeTypes for keys and values:
cache.getAdvancedCache.withMimeType("application/x-java-object", "application/json").withEncoding(IdentityEncoder.class)

```

Internally the cache will compare the passed in mimetype with the configured mimetype from the cache and will perform a transcode operation if they 
differ.

### Implementation

The transcode operation will take a tuple (Object, MimeType) plus a destination MimeType and will return a converted Object that will be sent to the encoder.

```java
import javax.activation.MimeType;

public interface Transcoder {

   /**
    * Transcode content between two different {@link MimeType}
    *
    * @param content         Content to transcode
    * @param contentType     The {@link MimeType} of the content
    * @param destinationType The target {@link MimeType} to convert to
    * @param binary          if true, produces byte[] as output
    * @return the transcoded content
    */
   Object transcode(Object content, MimeType contentType, MimeType destinationType, boolean binary);

   /**
    * @return the {@link MimeTypePair} that is supported by this transcoder.
    */
   MimeTypePair getSupportedMimeTypes();

}

```

Transcoders will be stored along ```Wrapper``` and the ```Encoder``` in the ```EncoderRegistry```, which will gain two new operations:

```java
encoderRegistry.registerTranscoder(...)

encoderRegistry.getTranscoder(Mimetype type1, MimeType type2)
````

#### Example

Consider a cache defined as:

```xml
<!-- value is defined the same way... -->
<key mime-type="application/x-java-object" encoder="org.infinispan.commons.dataconversion.IdentityEncoder"/>
```

A ```cache.getAdvancedCache.withMimeType("application/json").put("{1}","{a}")``` will internally do the following transformations:

```java
// Dataflow inside the cache, for illustrative purposes only
Transcoder jsonPojoTranscoder = encoderRegistry.lookupTranscoder("application/json","application/x-java-object")
Object k = JsonPojoTranscoder.transcode("{1}","application/json","application/x-java-object")
Object v = JsonPojoTranscoder.transcode("{a}","application/json","application/x-java-object")
cache.put(IdentityEncoder.toStorage(k), IdentityEncoder.toStorage(v))
```

### Example Scenarios


#### 1. XML documents accessible via multiple endpoints (Hot Rod and Rest)

Since the cache is going to be used in server mode, it will receive data as ```byte[]``` from the client, so we have two choices:
either storing the ```byte[]``` directly by doing:

```xml
<local-cache>
  <data-type>
     <!-- No encoder declared, so it will store whatever is passed in -->
     <key mime-type="application/xml"/>
     <value mime-type="application/xml"/>
  </data-type>
</local-cache>
```
or we can store XML as a String by doing:

```xml
<local-cache>
  <data-type>
     <key mime-type="application/xml" encoder="org.infinispan.commons.dataconversion.UTF8StringEncoder"/>
     <value mime-type="application/xml" encoder="org.infinispan.commons.dataconversion.UTF8StringEncoder"/>
  </data-type>
</local-cache>
```

The ```UTF8StringEncoder``` will take UTF-8 bytes and convert it to a String, and in case it receives string, it'll just pass it through.

With the configuration above, let's see how data would be read and written in each endpoint:

##### Writing from Rest

A typical REST client will send data specifying the content type, as in:

```
curl -X POST -H 'Content-type: application/xml' -d @file.xml http://localhost:8080/rest/cache/1
```

Upon receiving this request, the rest endpoint will pass the received data to the encoder directly since the request MimeType
matches the configured MimeType for the cache.

An alternative way of sending content would be via ```text/plain```

```
curl -X POST -H 'Content-type: text/plain' -d @file.txt http://localhost:8080/rest/cache/1
```

This would be received by the rest server, it'd look for a Transcoder between ```text/plain``` and ```application/xml```, 
do the transcoding, and pass the result to the encoder for storage.

##### Writing from Hot Rod

Hot Rod does not allow to send the data type in the request (yet) so we initially assume the data sent is the same 
MimeType of the cache configuration. The Hot Rod client currently allows to plug-in different marshallers, so by using 
an ```UTF8Marshaller```, it can convert String to ```byte[]``` and send it over the wire. The server would just pass 
it to the encoder.

##### Writing from Embedded mode

A custom java task can be run against the server and in this case it'd allow the user to interact with the cache directly, 
with no network layer involved. This usage offers more flexibility, as the user can request on-demand the wanted encoding.

Example:
```java
// Bypass all encoding and put in the stored format directly:
cache.getAdvancedCache().withEncoding(IdentityEncoder.class).put(...)

// Write a json
// Will cause a transcoding from json to xml first and then it will be stored directly
cache.getAdvancedCache().withMimeType("application/json").withEncoding(IdentityEncoder.class).put("{json}")
```

##### Reading via REST

The rest client will include an Accept header asking for the format it wants the data to be returned:

```
curl -X GET -H 'Accept: application/json' http://localhost:8080/rest/cache/1
```

Internally, in the server, this will involve the following operations:

```
Stored format ----(Encoder.fromStorage)---->  XML (string or byte[])  ----(JsonXMLTranscoder.transcode)----> Json (String)  ---> Client
```

##### Reading from Hot Rod

Clients will receive the ```byte[]``` back, as Hot Rod does not allow to request data in a specific format (yet) so we initially assume the data will be received without 
any transcoding

##### Reading from embedded

Same as for writing, embedded caches are free to specify any encoding on the fly

#### 2. Querying and Indexing using json

Infinispan's query engine supports two data formats: protobuf and java entities annotated with hibernate search.
By using transcoding capabilities, we can enable querying and indexing by dealing with only json data in the caller:

Example configuration:

```xml
<local-cache>
  <data-type>
     <!-- No encoder declared, so it will store whatever is passed in -->
     <key mime-type="application/x-protostream"/>
     <value mime-type="application/x-protostream"/>
  </data-type>
</local-cache>
```

When a request arrives to the REST endpoint in the form:

```
curl -X POST -H 'Content-type: application/json' -d @file.json http://localhost:8080/rest/cache/1
```

Infinispan will lookup for a transcoder that can convert between ```application/x-protostream``` and ```application/json```

and will pass the converted result (a protobuf) to the cache, enabling thus, the indexing to happen.


WARNING: It'll be slightly more complicated than that, as first we need to craft a .proto file for the server and register 
it.

At the same time, Hot Rod clients would deal with protostream directly. They will probably have message marshallers to convert to and 
from protobuf.  Once Hot Rod has the capability of asking for a particular MimeType, they would be able to get the json directly.



#### 3. Compressing server

It'd be also possible to transparently compress and de-compress data by using the right configuration:

```xml
<local-cache>
  <data-type>
     <!-- keys are Strings, values are text that will be stored compressed -->
     <key mime-type="application/x-java-object"/>
     <value mime-type="text/plain" encoder="org.infinispan.commons.dataconversion.GzipEncoder"/>
  </data-type>
</local-cache>
```

From a Rest point of view, any ```text/plain``` content sent to Infinispan will be directly sent to the encoder and it'll 
be compressed, and when reading data, it will decompress it on-the-fly.

From the Hot Rod server point-of-view, the content would need to be sent as a ```byte[]``` representing text. Until Hot Rod has
the capacity of describing the format it is sending, it can't just marshall the text and sent it; the encoder expect 
it to be a text, so it should be sent as a UTF8 byte[] and not a JBoss marshalled content.

Embedded caches have the full freedom to use the advanced cache API to read and write data in any format/encoding

