Infinispan Server Restructuring
===============================

Status quo
----------

Infinispan Server is currently based on the full WildFly distribution.
It is composed of the following additional components:

  * a JGroups subsystem which is mostly aligned with the WildFly with some additions (e.g. SASL integration)
  * an Infinispan subsystem, originally forked from the WildFly one, but much more complete in terms of feature coverage, with support for deployments
  * an Endpoint subsystem, unique to Infinispan Server
  * the Infinispan admin console
  * a CLI which extends the WildFly CLI with some additional convenience commands

We use the WildFly feature pack and server provisioning tools to create the Infinispan Server package based on the above subsystem and on any additional modules we require (including new versions of modules already supplied by WildFly).
We also remove some unnecessary modules based on the output of a Python script which computes module/subsystem dependencies.
Unfortunately the "trimming process" is not as effective as it could be because some of the modules, while not needed by Infinispan Server, are hard dependencies of other server modules. In particular we rely on the Datasource and Transaction subsystem which have a lot of dependencies.

Basing Infinispan Server on WildFly has a number of advantages and disadvantages.

Advantages

   * JBoss Modules combined with the service model provides module and service dependency tracking with concurrent loading
   * User deployment support
   * DMR with support for inspecting / modifying / persisting configuration at runtime, with propagation to all nodes in the domain, accessible via CLI, Console and remoting
   * Security realms with support for certificate stores, LDAP integration, Kerberos, etc
   * The subsystems can be packaged as a WildFly layer to provide newer features to WF deployments
   
Disadvantages

   * Integrating with the server configuration model adds a lot of overhead
   * Larger than needed footprint
   * Much of the functionality supplied by WildFly is unnecessary in container deployments such as OpenShift

Redesign proposals
------------------

For future development we want to pursue two paths: one which maintains compatibility with the current situation but resolves some of the above disadvantages and a more aggressive one tailored for containers

Plan A: Base the server on WildFly Core

   * Make the dependency on the transactions and datasource subsystems optional by providing our own internal implementations by directly integrating with Narayana and HikariCP
   * Still work with the transactions/datasource subsystem if they are present (to support the WildFly layering)
   * Need to still support deployable JDBC drivers
   
Plan B: Create an uber-jar slim server

   * Add some bootstrapping code 
     * no need for WildFly Swarm or Spring Boot
   * Configuration
     * use the embedded Infinispan schema
     * configure endpoints and other components via the global modules API
     * persist configuration changes using the existing configuration serializer
     * add runtime clustered configuration capabilities (store configs in an internal cache)
   * Security
     * Depend on Elytron for LDAP, Properties, Certificates, Kerberos, KeyCloak, etc integration
   * Custom user "deployments"
     * Load custom code placed in an extension directory (use JBoss Modules?)
     * Scan only at startup
     * use META-INF/services/ discovery
   * Management
     * Runtime management performed via JMX
     * Revamp the Embedded CLI
     * The web console can be integrated with Jolokia which exposes JMX MBeans as RESTful endpoints
  
