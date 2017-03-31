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
