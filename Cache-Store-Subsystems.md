While the cache store configuration machinery in embedded is easily extensible, adding new cache stores to Infinispan Server is much more complex because of the "monolithic schema" approach that WildFly imposes on subsystems. This means that, in order to add a schema-enabled cache store to server, we have to modify the Infinispan subsystem itself. This applies to both "Infinispan-owned" cache stores which live outside the main Infinispan repo and to custom cache stores.

The proposed solution is to  create a dedicated subsystem for "external cachestores", akin to how datasources are currently configured. Cache stores would be defined in this subsystem and referenced from the main Infinispan subsystem using a naming convention (think JNDI).

We would need to define an archetype subsystem which can be extended for the specific cache store configuration logic with little effort.

Here is an example of the configuration for such a subsystem:

    <subsystem xmlns="urn:infinispan:server:cachestore:cassandra:8.2">
       <cassandra-store name="mycassandrastore"
                        auto-create-keyspace="true" keyspace="TestKeyspace"
                        entry-table="TestEntryTable"
                        consistency-level="LOCAL_ONE"
                        serial-consistency-level="SERIAL">
           <cassandra-server host="127.0.0.1" port="9042" />
           <cassandra-server host="127.0.0.1" port="9041" />
           <connection-pool heartbeat-interval-seconds="30"
                        idle-timeout-seconds="120"
                        pool-timeout-millis="5" />
        </cassandra-store>
    </subsystem>

The subsystem would be responsible for parsing the configuration and 
registering an appropriate StoreConfigurationBuilder under a named 
Service within the server. We need to be careful about class loader 
visibility, but the datasource subsystem does something similar, so it 
should be possible.

The main Infinispan subsystem would need to be extended to be able to 
parse the following (simplified):

    <subsystem xmlns="urn:infinispan:server:core:8.2">
       <cache-container name="clustered">
         <distributed-cache name="default">
           <store ref="infinispan/cachestore/cassandra/mycassandrastore"
                  passivation="true" fetch-state="true" preload="true"
                  purge="false" />
         </distributed-cache>
       </cache-container>
    </subsystem>

It would also not be difficult, for symmetry, to support a compatible 
schema for the embedded use-case.

The cachestores would be distributed both as a simple jar for embedded 
use-cases and as a zip containing the necessary modules for server.
Note that this would not leverage the deployable cachestores machinery 
as subsystems need to be installed as modules.
