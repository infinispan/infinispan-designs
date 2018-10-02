Infinispan ServerNG
====================

Infinispan ServerNG is a reboot of Infinispan's server which addresses the following:

* small codebase with as little duplication of already existing functionality (e.g. configuration)
* embeddable: the server should allow for easy testability in both single and clustered configurations
* RESTful admin capabilities
* Logging using [JBoss Logging logmanager](https://github.com/jboss-logging/jboss-logmanager)
* Security using [Elytron](https://github.com/wildfly-security/wildfly-elytron)

# Layout

The server is laid out as follows:

* `/bin` scripts
   * `server.sh` server startup shell script for Unix/Linux
   * `server.ps1` server startup script for Windows Powershell 
* `/lib` server jars
   * `infinispan-server.jar` uber-jar of all dependencies required to run the server.
* `/server` default server instance folder
* `/server/log` log files
* `/server/configuration` configuration files
   * `infinispan.xml`
   * keystores
   * `logging.properties` for configuring logging
   * User/groups property files (e.g. `mgmt-users.properties`, `mgmt-groups.properties`) 
* `/server/data` data files organized by container name
   * `default`
      * `caches.xml` runtime cache configuration
      * `___global.state` global state
      * `mycache` cachestore data
* `/server/lib` extension jars (custom filter, listeners, etc)

# Paths

The following is a list of _paths_ which matter to the server:

* `infinispan.server.home` defaults to the directory which contains the server files.
* `infinispan.server.root` defaults to the `server` directory under the `infinispan.server.home`
* `infinispan.server.configuration` defaults to `infinispan.xml` and is located in the `configuration` folder under the `infinispan.server.root`

# Command-line

The server supports the following command-line arguments:

* `-b`, `--bind-address=<address>`
* `-c`, `--server-config=<config>`
* `-o`, `--port-offset=<offset>`
* `-s`, `--server-root=<path>`
* `-v`, `--version`

# Configuration

The server configuration extends the standard Infinispan configuration adding server-specific elements:

* `security` configures the available security realms which can be used by the endpoints.
* `cache-container` multiple containers may be configured, distinguished by name.
* `endpoints` lists the enabled endpoint connectors (hotrod, rest, ...).
* `socket-bindings` lists the socket bindings.

An example skeleton configuration file looks as follows:

```
<infinispan>

   <!-- Global security configuration -->
   <security>
      <security-realm name="ManagementRealm">
         <authentication>
            <properties path="mgmt-users.properties" relative-to="infinispan.server.config.dir"/>
         </authentication>
         <authorization map-groups-to-roles="false">
            <properties path="mgmt-groups.properties" relative-to="infinispan.server.config.dir"/>
         </authorization>
      </security-realm>
      <security-realm name="PublicRealm">
         <server-identities>
            <ssl>
                <keystore path="application.keystore" relative-to="infinispan.server.config.dir" keystore-password="password" alias="server" key-password="password" generate-self-signed-certificate-host="localhost"/>
            </ssl>
         </server-identities>
         <authentication>
            <properties path="application-users.properties" relative-to="infinispan.server.config.dir"/>
         </authentication>
         <authorization>
            <properties path="application-roles.properties" relative-to="infinispan.server.config.dir"/>
         </authorization>
      </security-realm>
   </security>
   
   <!-- JGroups configuration -->
   <jgroups transport="org.infinispan.remoting.transport.jgroups.JGroupsTransport">
      <stack-file name="udp" path="jgroups-udp.xml"/>
      <stack-file name="tcp" path="jgroups-tcp.xml"/>
   </jgroups>
   
   <!-- Cache containers -->
   <cache-container name="default" ...>
      <transport .../>
   </cache-container>
   
   <!-- Endpoints -->
   <endpoints>
      <hotrod-connector cache-container="default" socket-binding="hotrod" ... />
      <rest-connector socket-binding="rest" ... />
      <admin-connector socket-binding="management" security-realm="ManagementRealm" />
   </endpoints>
   
   <socket-bindings>
      <socket-binding name="hotrod" address="${infinispan.bind.address}" port="11222"/>
      <socket-binding name="rest" address="${infinispan.bind.address}" port="8080"/>
      <socket-binding name="management" address="${infinispan.bind.address.management}" port="9990"/>
      <socket-binding name="cluster" multicast-address="${infinispan.bind.address.multicast}" multicast-port="45700"/>
   </socket-bindings>
</infinispan>
```

# Logging

Logging is handled by JBoss Logging's LogManager. This is configured through a `logging.properties` file in the 
`server/configuration` directory. The following is an example:

```
loggers=org.jboss.logmanager

# Root logger
logger.level=INFO
logger.handlers=CONSOLE

logger.org.jboss.logmanager.useParentHandlers=true
logger.org.jboss.logmanager.level=INFO

handler.CONSOLE=org.jboss.logmanager.handlers.ConsoleHandler
handler.CONSOLE.formatter=PATTERN
handler.CONSOLE.properties=autoFlush,target
handler.CONSOLE.autoFlush=true
handler.CONSOLE.target=SYSTEM_OUT

formatter.PATTERN=org.jboss.logmanager.formatters.PatternFormatter
formatter.PATTERN.properties=pattern
formatter.PATTERN.pattern=%d{HH:mm:ss,SSS} %-5p [%c] (%t) %s%E%n
```
# Internals

The following is a dump of various internal facts about the server, in no particular order:

* All containers handled by the same server share the same thread pools and transport.
* When a server starts it locks the `infinispan.server.root` so that it cannot be used by another server concurrently.
* The `admin-connector` endpoint is a special type of `rest-connector` with additional ops
* The CLI connects to the `admin-connector` using either
   * the `local-user` SASL mech provided by Eltryon when running on the same host/user
   * any HTTP auth supported by the rest endpoint
   
 
