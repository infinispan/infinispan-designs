# Operator Migration Upgrade
Optionally allow a Infinispan CR cluster to be upgraded with zero downtime and no loss of data.

## Spec
Add a field to the CR spec that determines whether an upgrade should be `migration` or `gracefulShutdown`.

```yaml
apiVersion: infinispan.org/v1
kind: Infinispan
metadata:
  name: example-infinispan
spec:
  upgrades: migration
```

## Procedure
When an upgrade is possible, the operator performs the following for each Infinispan CR:

1. Create a new StatefulSet for the target cluster based upon the CR spec, wait until the cluster is formed

2. Retrieve all cache configurations from the cluster in the original Statefulset

3. Append a remote-store configuration to each of the ^ configurations and create cache target cluster:

```xml
<remote-store xmlns="urn:infinispan:config:store:remote:12.0"
                 cache="<cache-name>" 
                 protocol-version="<protocol-version>"> 
      <remote-server host="<ip-of-pod>" port="11222"/> 
   </remote-store>
```

4. Switch clients to the target cluster.

5. Update the CR status.StatefulSetName to new Statefulset

6. Sync data from source for all caches. Execute the following on the target cluster

```
POST /v2/caches/{cacheName}?action=sync-data
```

7. Disconnect remote-store on the target cluster for all caches
```
POST /v2/caches/{cacheName}?action=disconnect-source
```

8. Delete the source StatefulSet

## Switching Clients
- How do we ensure that the single-port service only points to the target cluster?

## Xsite
?
