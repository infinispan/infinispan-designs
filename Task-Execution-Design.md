Some of the most powerful features in Infinispan involve execution of code "close to the data": distributed executors and map/reduce tasks allow running user tasks using the inherent data parallelism and the computing power of all the nodes in a cluster.
While it is interesting to allow running such tasks from HotRod, the concept can be extended even further to the execution of any kind of code, be it local to a single node or distributed among some/all the nodes in the cluster.
In the spirit of HotRod language-agnostic nature, we should not limit execution just to Java, but leverage the JDK's scripting engine API so that tasks can be developed in a multitude of languages.

# Stored scripts 
* Store scripts in a dedicated script cache (persistent, secure, etc)
* If the scripting engine supports it, precompile to bytecode to take advantage of HotSpot
* Each script can take multiple named parameters which will appear as bindings (i.e. variables) in the script
* Script bindings include the cacheContainer, cache, scriptingManager, marshaller (if in HotRod)
* A script can optionally return a result
* Scripts can be of various types supported by JSR-223 (Java, Javascript, Scala, etc)

# Remote execution over HotRod
* Add EXEC op
* Input parameters: (name: string, value: byte[])* < marshalled values
* Returns: (byte[])?
* Optional flags to specify execution type (local, distributed, map/reduce)

# Remote execution over REST
* Invoke a POST on the URL http://server/rest/!/scriptname/cachename?param1=value1&param2=value2
* Invoke a POST on the URL http://server/rest/!/scriptname/cachename and pass any parameters in the request body.
* Manipulate sync/async execution using the 'performAsync' header used by other ops
* The returned value will use the request variant to perform conversion appropriately to the requested type (text/plain, application/octet-stream, application/xml, application/json, application/x-java-serialized-object)

# Task manager
* The task manager will be the main entry point for executing tasks and for retrieving execution history
* Task execution is delegated to specific engines
** Scripts will be handled by the existing ScriptManager
** Server-deployed tasks will be handled by a DeployedTaskManager
* The executor within which tasks are executed is global to all engines.
* Track start/finish/what/who/where for tasks that have been executed. Store these in an internal, persistent cache which can be queried/filtered appropriately. Possibly have a limited retention.
* Support aborting running jobs (best-effort, since it requires handling of InterruptedException in appropriate places. Badly-behaved tasks will not be stoppable until JVM termination).
* Throttling ? (via some executor / thread priority modification)

