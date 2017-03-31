#Summary

Use the total order protocol to handle commands in non transactional cache.

#Changes in configuration:

* `transaction-protocol` attribute will be removed from `<transaction>` tag.
* `TOTAL_ORDER` option will be added `locking-mode`.

#Description (for a normal scenario):

A client invoked a write operation in node `N`:
* `N` sends a total order message to all the owners with the write operation;
* All owners deliver the write operation and perform it;
* `N` waits for all the replies and returns the result back to the client;

**Why it works?**

Since the write operations are deliver in total order by all the owners of the key(s), then when the write operation is deliver, it sees the same state and performs the same steps, generation the same new state.

**Note:**
In the initial version, the commands are processed directly in the JGroups thread. If it justifies, a new non-transactional total order manager may be created and the operations start to be processed in the remote executor service.

**Optimizations:**
The originator does not need to wait for all the replies. The reply can be marked as `STABLE` or `UNSTABLE` (or other name; the goal is to mark as unstable the replies from commands processed during the state transfer). Then, the originator waits:

1. for the first reply is the reply is marked as `STABLE`, or it is an exception, or `IGNORE_RETURN_VALUE` is set and the operation is non-conditional;
2. for the first non-null reply when the reply is marked as `UNSTABLE`.

#State Transfer

As in total order transactional caches, the cache topology commands which modifies the topology ID are sent in total order. Having this in mind, we have:

* **when the command topology ID is different from the current topology ID**

Exception is throw. Originator re-sends the operation.

**Why?**

If the operation is delivered in a different topology ID, then the new owners will not see it, breaking the data consistency.

**Can't it forward the operation to the new owners?**

No. No order is guarantee when forward operations. Imagine the following scenario:

1. Node `N` sends two operations `C1` and `C2` for the same key. `C1` is sent in topology `i` and `C2` in topology `i+1`.
2. `C1` and `C2` are delivered in topology `i+1` and `C1` is delivered before `C2`. 
3. If `C1` is forward, the new owner could delivered the operation `C1` following by `C2` or `C2` following `C1`. If the later happens, the data consistency is broken and the algorithm too.

**optimization:**
In replicated caches, if only nodes are leaving, the operation can be processed normally.

* **when a conditional operation is delivered and state transfer is in progress**

Exception is throw. Originator should block the operation until the state transfer is finished.

**Why?**

When the operation is delivered in the new owners, they don't have the current value to check the condition.

**Can't it fetch the current value from old owners?**

No. The remote get is not ordered with the operation, so it can return a value after or before the operation. If a value before the operation is returned, the consistency is broken.

**Can't the command wait in the new owner until the state is received?**

No. It is going to block the total order deliver thread and it will never deliver the new topology ID.

**optimization:**
if the key is not affected by the state transfer (i.e. the ownership didn't change), then the conditional operation can be processed normally. However, it causes the following:

1. in replicated caches, a node joining will block all the conditional operations;
2. in replicated caches, a node leaving does not block any operations.

* **when a non-conditional operation is delivered and state transfer is in progress**

The command is processed normally and the originator takes the replies and return the value different from `null` (if any).

**Why?**

First, the operation does not dependent on the state of the data. Second, the old owners will return the current value when the operation is deliver and the new owner can return `null` or the current value (if it was already received from state transfer).