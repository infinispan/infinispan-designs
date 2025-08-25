# Operator Inline Upgrade
Optionally allow a Infinispan CR cluster to be upgraded with zero downtime, no loss of data and minimal extra resources usage.

## Spec
Add an new value `inline` to the `spec.upgrades` field.

```yaml
apiVersion: infinispan.org/v1
kind: Infinispan
metadata:
  name: example-infinispan
spec:
  upgrades: inline
```

## Procedure
This is a specialization for then Infinispan Operator of the general JGroups procedure described in [JGroups RollingUpgrades docs](https://github.com/jgroups-extras/RollingUpgrades/)  
When an upgrade is required the operator performs the following procedure:
1. Enable the JGroups UPGRADE protocol on the existing cluster
2. Deploy an UpgradeServer
3. Enable/add the UPGRADE protocol
4. Start an Infinispan node with the new configuration
5. Wait for the cluster view to be well formed
6. Kill an old node
7. Loops from 4 until all the old nodes are gone
8. Disable the UPGRADE protocol
9. Kill the UpgradeServer

## Notes
- This procedure will require the operator to manage two different StatefulSets at the same time and switch from the old one to the new at the end of the upgrade.
- All the ready nodes should be able to process user requests.
- `WellFormed` condition will probably need a review.
- X Site?


