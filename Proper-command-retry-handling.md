This document summarizes changes introduced in [ISPN-7590](https://github.com/infinispan/infinispan/pull/5064).

# Goals
* Command should never be applied to an entry twice. 
* If a command does not ignore previous value (on backup), it must be applied on the backup to the same value as on primary.
* The return value of a command should be reflect the change regardless of retries (e.g. replace will return old value even if it is executed multiple times)
* Events should have correct type (created/modified)
* Events should be fired on each node only once

# Non-goals
* Correct previous value in events - events won't be reliable anyway as some commands don't load previous value from persistence/remote node etc.
* Solve [ISPN-3918](https://issues.jboss.org/browse/ISPN-3918) - although this was amongst the goals initially, the fix would have negative performance impact or would be out of scope (flag-affected behaviour).
* Fix anything related to functional events
* Deal with delta-awares more than what is required to pass current testsuite

# Concepts
* The fact that a command has been applied to a node is stored as `InvocationRecord` in metadata.
  * Invocation records expire over time; this is executed by the expiration thread. The default expiration time is 4 times the replication timeout, but if an attempt is made to apply the value again, the expiration is postponed.
  * When a command is completed on the originator, the (key, command-invocation-id) pair is sent to background process (in `InvocationManager`) which invalidates he invocation records in a batch.
  * As the invocation record has to be held in metadata even for removed entries, both data container and cachestores have to be able to hold entries with null value (aka tombstones).
  * Since backup may not know previous value of an entry when the command is applied (replace is non-conditional on backup owner when it succeeds on primary and does not have to load previous value), the return value on primary has to be stored in the invocation record for the case that this backup becomes the next primary. From reasons above, primary has to replicate return value of a command to all backups, too.
* `InvocationManager` keeps information about invalidated invocations per segment, and once this exceeds a threshold (for given segment) - 100 by default, customizable through `infinispan.invocations` property ATM - sends `ForgetInvocationsCommand` to all write owners of given segment. Non-current owners are expected to drop the invocation records when these lose ownership of the segment.
  * In async modes we can't do any invalidations because originator does not know when the command finishes; we have to rely on expiration. However, we have to store the invocation record even if the entry is committed with `ctx.isOriginLocal()`. Regrettably this forces us to store the information about command being executed in sync/async mode in the command itself.
* Commands that require the same previous value on backup as on primary implement `StrictOrderingCommand`. At this point these are ReadWrite* functional commands. When a strict command is replicated to backup, primary adds command invocation id of previous command (or null if there's no previous record). The backup checks its last record against this last invocation id and applies the command only in case of match. In other case it retrieves full entry including metadata with invocation records from the primary owner, and stores this value (potentially applying the command it it was not applied - this depends on classical/triangle routing algorithms).
  * Alternative comparison could be based on version numbers, but a) not all configurations use versioning b) version numbers are reset when the entry is removed - this would be another intrusive change.
* When command is found to be already applied, it should be marked with `WriteCommand.setCompleted(key)`. Persistence layer shouldn't store the entry second time (it should check `WriteCommand.isCompleted(key)`).
* In transactional mode, we keep the invocation records in the context to make sure we don't apply the record twice when the prepare is retried, but we don't commit those into datacontainer/cachestores.
  * There were some changes related to versioning of non-existent entries which result in VersionReadInterceptor in tx versioned modes.

# Other info
* Value matchers are completely gone :)
* When the primary finds out that it has the command applied, it does not apply it second time but has to replicate it to all backups.
* The authoritative flag in invocation record was included as an attempt to solve ISPN-3918. However the initial idea does not work with triangle.
* EvictCommand and InvalidateCommand/InvalidateL1Command don't store invocation records. ClearCommand neither. 
* DataContainer.containsKey() returns false if there's just the tombstone, but DataContainer.get() returns the entry (with null value and non-null metadata).
