This wiki describes the design of the cross-site failover for Hot Rod clients ([ISPN-5520](https://issues.jboss.org/browse/ISPN-5520)).

Hot Rod clients currently support failover to nodes within the cluster to which the client is connected. To support that, Hot Rod servers send topology information to clients as part of the responses as topology changes happen. When an operation to a clustered node fails due to transport or cluster rebalancing issues, the client automatically retries the operation in a different cluster node. This node is elected using a configured load balance policy.

To add basic cross-site failover support, the following changes are required:

* Client configuration needs to be enhanced to have 0-N cross-site static configuration, where each cross-site configuration would have 1-N host information of nodes in that site.
* In its simplest form, a Hot Rod client should failover to the nodes defined in cross-site failover section, if all nodes in the main cluster have failed (after a complete retry). The first site where nodes are available becomes the the client's main site, working as usual with the topology of this site.

Cross-site failover could be improved further if clients would be able to failover when the site is offline while still being accessible remotely. To achieve this, there'd need to be a way for the server to tell the clients that the site is offline. The simplest way to do so would be to add a new response status that indicates that the site is offline, so next time the client sends an operation and gets offline status, it fails over to a different site.