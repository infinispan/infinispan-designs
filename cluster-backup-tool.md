Cluster Backup Tool
====================
# Requirements
## Hard
- Ability to:
    - Backup data to a file(s) at a given point in time
    - Recreate data from archive
- Must be backwards Compatible across x major versions
- Supported in client/server mode

## Optional/Future
- Topology aware restoration
    - Disabled by default, focus is on data

- Import failure modes:
    - Clusterwide: Rollback all on first error
    - Per Cache: Only rollback importing of caches which encounter errors

- Cache specific imports
    - Greater flexibility
    - Ability to retry failed cache imports
    - Convert from one cache type to another

- Configuration only backup/import

- Enable "online" backups where the cluster creates a snapshot from a live cluster
    - Utilise `CommitManager` as used by state transfer and conflict resolution
    

# Architecture
The backup tool should be implemented on the server and be accessible via the `/v2/cluster` REST endpoint, allowing the various
Infinispan clients (CLI, Console, Operator) to utilise it. Backups can be initiated via a GET call and restored by
uploading an archive via a POST.

Backup:
```
GET /v2/cluster?action=backup
```

Restore:
```
POST /v2/cluster?action=restore
Content-Disposition: form-data; name=`"file`"; filename=`"backup.zip`"
Content-Type: application/zip
Content-Transfer-Encoding: binary
```

# Archive Format
A cluster backup consists of a global manifest which contains clusterwide metadata, such as the version. As well as a
directory structure that contains all cache templates, cache instances and counters. These files are then distributed
as a archive format, which can be passed to the CLI in order to initiate a import.

```
cache-configs/
cache-configs/some-template.xml
cache-configs/another-template.xml
caches/
caches/example-user-cache.dat
caches/example-user-cache.xml
counters/
counters/counters.dat
manifest.properties
cache-container.xml
```

The above files will be packaged as a single `.zip` distribution to aid backup/import; this provides compression as well
as being compatible with both Linux and Windows.

## Manifest
Basic java properties file with key/values.

```java
version=11.0.0.Final
cache-configs="some-template, another-template"
caches="org.infinispan.CONFIG", "example-user-cache"
counters="example-counter"
```

The Infinispan version can be used by the Server to quickly determine if a migration to the new server version is possible
based upon the number of major versions we support migrations across.

## Container XML
The container XML can be used to determine if the backup depends on any additional user classes, e.g. serialization marshallers,
and can check that the server contains these resources on it's classpath, failing fast if they are not present.

## Template files
Each defined template is represented by a `<template-name>.xml` file.

## Cache Files
A cache backup consists of a `<cache-name>.xml` and a `<cache-name>.dat` file. The `.xml` file contains
the cache configuration and is used to create the initial empty cache. The `.dat` file is a batch file containing the cache
content and is read by the server in order to restore entries to the cache. Each line in the file corresponds to a cache
entry, with keys and values separated by a single space.

> If the cache is empty during the backup, then no `.dat` file is created.

Example  `.dat` content:
```java
hello world
hola mundo
```

> How to represent metadata and handle expiration?

### Storage MediaTypes
If a cache has a configured key and/or value MediaType, that can be readily represented as a string, e.g. text/plain or json,
then the key/value should be represented in the `.dat` file using that format. Otherwise a binary MediaType, such
as `application/x-protostream`, are represented as a Base64 String.

Example  `.dat` content:
```java
hello V29ybGQ=
```

### Internal Caches
Volatile internal caches should not be included in a backup. Internal caches can be excluded during the creation of the
backup archive, with no `.xml` or `.dat` files created.

The counter cache should also be omitted in favour of a dedicated `counters.dat` file.

## Counters File
The `counters.dat` file that contains the informated required in order to recreate all counters and their values
at backup time. If no counters exist, this file can be omitted from the archive.

```
<NAME> <STORAGE> <TYPE> <INITIAL_VALUE> <CURRENT_VALUE>
example-cache PERSISTENT STRONG 0 5
```

# CLI Integration
The CLI will be the first client to expose the backup/restore capabilities, proposed syntax:

Backup:
```bash
bin/cli.sh -c localhost:11222 --backup <archive-file>
```

Restore:
```bash
bin/cli.sh -c localhost:11222 --restore <archive-file>
```

# Missing Capabilities
List of missing features that are a prerequisite for the backup/restore feature.

## Endpoints
* Ability to reject all requests in backup mode?
