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
The backup tool can be implemented as an extension to the CLI, with existing cache data stored as the corresponding CLI
commands required to add the data to a cache. A cluster backup consists of a single global manifest which contains clusterwide
metadata. As well as `.json` and `.dat` files for each cache and a single `counters.dat` file. These files are then
distributed as a archive format, which can be passed to the CLI in order to initiate a import.

Backup:
```bash
bin/cli.sh -c localhost:11222 --backup <archive-file>
```

Import:
```bash
bin/cli.sh -c localhost:11222 --restore <archive-file>
```

## Global Manifest
Contents:
- Infinispan Version used to create backup
- Global configuration
- Cache names

The Infinispan version can be used by the CLI to quickly determine if a migration to the new server version is possible
based upon the number of major versions we support migrations across. Similarly, the Global configuration can be used to
determine if the backup depends on any additional user classes, e.g. serialization marshallers, and can check that the
server contains these resources on it's classpath, failing fast if they are not present. Finally, the cache metadata is a
map of all cache names and their associated import file that are contained in the backup.

> A key assumption is that in order to import a data backup to Infinispan version `x`, it's necessary for version `x` of
the CLI to also be used.

### Format
Basic json file with key/values, allowing for future changes to ordering and for the GlobalConfiguration json to be
nested.
```json
{
	"version": "10.1.1.Final",
	"global-configuration": "...",
	"caches": ["org.infinispan.CONFIG", "example-user-cache"],
    "counters: ["example-counter"]
}

```

## Cache Files
A cache backup consists of a `.json` and `.dat` file, each of which is named after the cache. The `.json` file is a json
representation of the cache's configuration and the `.dat` file is the batch file executed by the CLI as part of the import.
Below is an example `.dat` file for a cache with the name ``example-user-cache`:

1. Create the cache using the `.json` file:

```
create cache --file=example-user-cache.json example-user-cache
```
2. Process the `example-user-cache.dat` batch file:

```
cd caches/example-user-cache
put hello world
put hola mundo
```
> If the cache is empty during the backup, then no `.dat` file is created.

### Internal Caches
Volatile internal caches should not be included in a backup. Internal caches can be excluded during the creation of the
backup archive, with no `.json` or `.dat` files created for the blacklisted caches.

The counter cache should also be omitted as CLI import statements in the `counters.dat` file should be used instead. 

### Storage MediaTypes
If a cache has a configured key and/or value MediaType, that can be readily represented as a string, e.g. text/plain or json,
then the `cache put` batch commands should utilise that encoding. Otherwise a binary MediaType, such
as `application/octet-stream` is used and the key/value represented as Base64.

For example, the following batch command would be used to store an try in a cache with keys set to `text/plain` and value
`application/json`:
```
cache put --encoding=application/json '{"some":"value"}' key
```

Or to put an entry with unknown key/values
```
cache put --key-encoding=application/octet-stream --value-encoding=application/octet-stream a2V5Cg== c29tZSB2YWx1ZQ==   
```

## Counters File
The `counters.dat` file that contains the CLI commands required in order to recreate all counters and their values
at backup time. If no counters exist, this file can be omitted from the archive.

```
create counter --initial-value=<value> --storage=<PERSISTENT|VOLATILE> --type=<STRONG|WEAK> <name>
```

## Backup Archive Format
The backup files should be packaged as a single archive distribution to aid backup/import.

> Is `.zip` sufficient?

> Should we make this optional to allow for greater flexiblity. Default to directory backup? Optional --compress flag?

# Missing Capabilities
List of missing features that are a prerequisite for the backup/restore feature.

## CLI
* Encoding:
    - Ability to specify --key-encoding and --value-encoding separately
        - Ideally it would be possible to specify once per batch, with subsequent puts just requiring `put <key> <value>`
    - Enable non text/plain values to be put inline
        - e.g `cache put --key-encoding=application/octet-stream --value-encoding=application/octet-stream a2V5Cg== c29tZSB2YWx1ZQ==`

## Rest
* Ability to iterate over cache entries

## Endpoints
* Ability to reject all requests in backup mode?
