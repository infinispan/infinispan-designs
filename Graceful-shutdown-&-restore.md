## Purpose
Implementing graceful shutdown and restore is essential for a datagrid which needs to be "cycled" for maintenance:
* shutdown: stop all nodes in a cluster in a safe way by persisting state and data
* restore: restart the whole cluster, apply previous state and preload the data from the storage (no data loss)

### State configuration
State configuration should be done at the GlobalConfiguration level. It should include the following settings:
* a state directory, where state will be persisted
* the expected number of nodes in the cluster. State transfer will not initiate until all nodes have joined the cluster. The default is 0, which corresponds to the current behaviour.

### Persistent State
The persistent state will be stored in the state directory. State should consist of multiple state files: 
* Global State File (___global.state)
  * localUUID
* Per-cache State File (cachename.state)
  * hash.function = hash class name
  * hash.numOwners = number of owners in the hash
  * hash.numSegments = number of segments in the hash
  * hash.members = number of members in the hash
  * hash.member.n.uuid = the UUID of each member
The state file will use the standard Java property file format for simplicity

### Startup sequence
    if statedir != null {
        check for state existence stored in statedir
        set UUID of local node to stored value
        start transport
        wait for expected view
        coordinator pushes configuration to all
        send consistent hash
        potentially stagger cache starts to avoid storm
    } else {
        start transport
        if !coordinator {
            send join request
        } else {
            wait for numInitialMembers to join
            send consistent hash
        }
    }

### Shutdown sequence
    for each running cache {
        put cache in shutting down state
        disallow new local ops/txs
        optionally wait for pending ops/txs
        send "ready to shutdown" to coord
        coord sends ack and does not process any more CH updates
        flush stores (passivation / async)
    }
    save state
