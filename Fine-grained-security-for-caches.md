We would like to enable fine-grained security to data within caches.

Two approaches:

## Authorization Callback
Via some kind of user-provided callback which would intercept all calls. The callback would receive the key, value and the required permission (READ, WRITE, etc) and allow/deny the operation. 

### Pros
* no additional memory is required to store the ACL
* can implement per-user checks. 

### Cons
* requires custom code.

Proposed interface:

```
public interface AuthorizationCallback {
   void authorize(Subject subject, Cache<K, V> cache, K key, V value, AuthorizationPermission permission) throws SecurityException;
}
```

Global cache operations (i.e. execute, listen, etc) would still go through the existing security checks.

## Authorization metadata
Store ACLs within each entry's metadata. This would require adding an additional field to the metadata to store the ACL information. The ACL would consist of:
* the owner of the entry. This needs to be as compact as possible while preserving uniqueness, and we need some kind of global mapper which can extract this information from the source of user information (e.g. a UID, GUID, etc)
* a set of authorization roles (as already defined for the container). Each role would imply specific permissions, so that a user would be allowed access/manipulation of an entry only if it is the owner, an administrator, or it belongs to one of the roles.

Pros
* does not require custom code (i.e. avoids having to deploy code to server)
* for the basic use-case (only the owner and the admin can manipulate an entry) nothing else needs to be done aside from enabling. A special ownership manipulation API would need to be defined for other cases.

Cons
* requires additional memory per entry. This should be mitigated using a BitSet.
* the granularity is per-role and not per-user
* ownership manipulation over remote needs protocol changes

Proposed API:

```
interface AuthorizationManager {
...
Set<String> getEntryRoles();
void setEntryRoles(K key, String... role); // this operation can be performed either by the owner or an ADMIN
}
```
