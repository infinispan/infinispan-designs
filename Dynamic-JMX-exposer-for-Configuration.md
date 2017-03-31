# Introduction

Currently configuration values are exposed in JMX as a summary list (see CacheImpl#getConfigurationAsProperties). This approach covers only read-only access and some of the Configuration's attributes could have been modified in runtime. This requires a new approach - scanning and exposing Configuration attributes dynamically.

More information might be found in tickets:
* https://issues.jboss.org/browse/ISPN-5343
* https://issues.jboss.org/browse/ISPN-5340

# User Perspective

Configuration attributes will be exposed as a new Node in JMX MBean tree on the same level as Activation, Cache, LockManager etc. 

# Implementation details

A new Dynamic MBean [1] will be created as a standard Infinispan component and will have a reference to Cache Configuration. During the startup it will scan the Configuration recursively and construct a `Map<String, ReflectionInfo<Class, Field>>`. The map's key will contain path to the attribute, for example: `clusteringConfiguration.asyncConfiguration.replicationQueueInterval` and the map's value will have all necessary information for making reflexive calls (a class and a field and possibly some additional metadata). Those attributes will used to provide MBean metadata and perform reflexive calls in the runtime.

The newly created Dynamic MBean will be discovered by `ComponentsJmxRegistration` and will be registered in MBean Server. Note that currently this class uses `ResourceDMBean`s for this, so additional scanning for Dynamic MBeans would have to be performed there.

# FAQ

Q: Shouldn't we use `ResourceDMBean` directly?

A: No. `ResourceDMBean` implementation is focused on invoking Managed Operation on a single class level. We need something more - we need to scan a bunch of configuration classes recursively.

Q: Shouldn't we prepare metadata when parsing classes (using `ComponentMetadataPersister`)?

A: We could but the parsing logic would have to fit into `ResourceDMBean` then and we would need to do a lot of code refactoring to support it. I'm not sure if it's worth it.

[1] http://docs.oracle.com/cd/E19698-01/816-7609/6mdjrf83d/index.html