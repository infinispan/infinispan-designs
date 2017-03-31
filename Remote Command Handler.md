# Summary

In clustered caches, the data is always replicated to other nodes. When a node receives a command, it is processed immediately. However, this approach have some issues because the commands are not independent among them (they can depend on other commands or conflict) that can make the thread enter in a waiting state. The worst scenario occurs when all threads are waiting for an event and none is available to process this notification. In this scenario, the cluster blocks and exceptions start to be thrown (usually `TimeoutException`).

# Goal 

Implement a "smarter" algorithm to handle remote commands. The algorithm will order the commands in order to minimize conflicts and waiting state between them.

# Algorithm

In the following, it explains how the remote commands are handled. A command is considered "blocking" if it needs to wait for an external event, such as, lock release, reply from other node, etc. 

## Introducing the `RemoteLockCommand`, the `RemoteLockOrderManager` and the `LockLatch`

The `RemoteLockCommand` is the interface that the `RpcCommands` must implement to be handled by this algorithm. This interface has a single method to return the key(s) to acquire lock, `Collection<Object> getKeysToLock()`.

The `RemoteLockOrderManager` main goal is to order the commands. Two or more commands conflict if they update in one or more common key(s). Since the lock can only be acquired by a single command, the `RemoteLockOrderManager` will define which command advances while the others wait. The interface only has a single method, `LockLatch order(RemoteLockCommand)`.

The `LockLatch` is a latch that notifies when the command can be processed. Also, it notifies when the command is done with its work and unblocks other commands. It has two methods:

* `boolean isReady()`: returns `true` when it is probable that the `Lock` is released.
* `void release()`: must be invoked after its work is done and the `Lock` is released. This will notify waiting command and may unblock other commands.

Finally, when a command is ready, it is sent to the remote executor service.

## How the handling is made?

### Read Command

No changes are made. When a read command is delivered, it is processed directly in the JGroups executor service.

### Non-Transactional Cache

We have two types of write commands: the ones delivered in the primary owner, which will acquire the lock and it sends the update to the backup owners; and the ones delivered in the backup owner. Since the later ones does not block, they are processed immediately in the JGroups executor service. The first ones, are managed by the `RemoteLockOrderManager` and processed in the remote executor service.

**State Transfer**
No changes needed. If the command if processed in the wrong topology id, an exception is thrown and the command is retried. The `LockLatch` and the `Lock` are released before the retry.

**Update: 16/09**
Needs to be implemented for the `PutMapCommand` and the `ClearCommand`. They will be similar to the description above, but the `LockLatch` implementation will be a collection of single key `LockLatch`es.

### Transactional Cache

**Update: 16/09**
Not implemented yet...

#### Pessimist Caches

In pessimistic caches, the algorithm handles the following commands:

* `LockControlCommnad` is managed in `RemoteLockOrderManager` and the `LockLatch` is associated to the transaction. Later the command is processed in the remote executor service.
* `PrepareCommand` is processed directly in JGroups executor service if L1 is **disabled**. In this case, it will not acquire any locks neither wait for any other events. But if L1 is **enabled**, it is processed in remote executor service because it needs to invalidate the L1 (synchronous operation). Note that it isn't managed by the `RemoteLockOrderManager` and it releases the `LockLatch`es associated to the transaction.
* `RollbackCommand` processed directly in JGroups executor service andit releases the `LockLatch`esassociated to the transaction.

**State Transfer**
It needs some mechanism to create `LockLatch` when the transactions transferred. Note that the `LockLatch` does not need to send to the other node, but the transaction needs to create open `LockLatch` in the new primary owner. _(to think better about it)_

#### Optimistic Caches

In optimistic caches, the algorithm handles the following commands:

* `PrepareCommand` it is managed by `RemoteLockOrderManager` and are processed in the remote executor service. The `LockLatch` is associated to the transaction.
* `CommitCommand` processed directly in JGroups executor service when L1 is **disabled**. When L1 is **enabled**, it is processed in remote executor service since it needs to invalidate the L1 (synchronous operation). Also, it releases the `LockLatch`es associated to the transaction.
* `RollbackCommand` processed directly in JGroups executor service and it releases the `LockLatch`es associated to the transaction.

**State Transfer**
The same as in pessimistic caches _(think...)_.

# Changes in default configuration

The default configuration uses a non-queued executor service with a large number of thread. With this new algorithm, it would be better to have a large queue and a reasonable number of threads.

Also, it would be good to merge the total order executor service with the remote executor service. In the end, they both have the same goal and it is not needed to configure two executor services. 

# Known Issues

If the remote executor service queue is not large enough to process all the "blocking" commands, it can hit the same problem again. The algorithm assumes that "blocking" commands are processed in remote executor service and the remaining in JGroups executor service.

An idea to solve this, is the executor service to expose their internal state (running threads and queue occupation). This way, it would make it possible to the algorithm to queue a bit longer the commands in order to prevent an overload of the executor service.

## L1 invalidation deadlock

When L1 is enabled, each update (or transaction) will generate an invalidation message to invalidate the L1 cache in non-owners. 

**non transactional caches**
It has at least 3 messages to be processed in the same pool. The message sent to the primary owner to lock generates an invalidation message and a message to the backup owners. At the same time, the backup owners generate another invalidation message. when the invalidation is synchronous, the executor service may be full with threads awaiting the ACK from invalidation (or from the backup owners) and no threads are available to process other requests.

**transactional caches**
The commit generates an invalidation message. If the executor service is full, the same problem as described above can occur.

## State Transfer Forwarding

This problem happens only with transactional caches. When a topology changes, the nodes involved in the transaction will forward the command (prepare and commit) for the new owners. Since the forward commands are processed in the same executor service and the "non-forward" commands, it can happen the executor service to be full.