Spring 5 will have a couple of [interesting features](https://jira.spring.io/browse/SPR-13574?jql=project%20%3D%20SPR%20AND%20issuetype%20%3D%20%22New%20Feature%22%20AND%20fixVersion%20in%20(%225.0%20M2%22%2C%20%225.0%20M3%22%2C%20%225.x%20Backlog%22)%20ORDER%20BY%20issuetype%20DESC):

* [CDI support (JSR 330)](https://jira.spring.io/browse/SPR-12211). We may enhance our CDI Extension to work correctly with Spring and propose this configuration for our users. Even though our extension has no hard dependency to Weld (apart from tests) I believe we will need to adjust it a little bit.
* [JCache 2.0 support](https://jira.spring.io/browse/SPR-13574). Implementing this issue will probably require adding few bits into Spring module and verifying if everything works smoothly.

A few ideas not necessarily related to Spring 5:
* Implement a "bridge" between [Spring tasks](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#scheduling) our [Infinispan distributed tasks](http://infinispan.org/docs/dev/user_guide/user_guide.html#DistributedExecutor).

Scheduled bigger features:
* [Spring Data-like use case](https://issues.jboss.org/browse/ISPN-1068)
* [Spring starter](https://issues.jboss.org/browse/ISPN-6561)
* [Define how to specify configuration](https://issues.jboss.org/browse/ISPN-3337)
