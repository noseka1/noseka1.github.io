+++
title = "Open Policy Agent, Part III — Integrating with Your Application"
date = "2019-12-03"
slug = "2019/12/03/open-policy-agent-part-iii-integrating-with-your-application"
Categories = [ "development" ]
+++

In the [previous entry](/blog/2019/10/27/open-policy-agent-part-ii-developing-policies/) to this series, we discussed developing policies with Open Policy Agent. In this final article in the series, we are going to focus on how you can integrate Open Policy Agent with your application.

<!--more-->

## Integrating OPA with your application

There are several options how you can integrate OPA with your application. If you happen to build your application using the Go language, you can link OPA as a library straight into your application. Otherwise, you will run OPA as a stand-alone service (daemon). If you deploy your application on Kubernetes, you can run OPA service as a side-car container along with your application services. This minimizes the communication latency between OPA and your application. It also avoids possible communication issues between OPA and your application due to network failures. If you are deploying on virtual machines, you can run one replica of the OPA service on each of your virtual machines to achieve the same benefits. In summary, deploying OPA as a side-car service or a host-local service is the recommended approach.

{{< figure src="/images/posts/open_policy_agent/opa_integration.png" class="center" >}}

Another deployment option would be running multiple OPA services behind a load balancer. Your application and OPA services would in this case talk over the network and the communication would go through the load balancer. You would incur the cost of network latency. However, I am not sure how would the overall reliability of this approach compare to the side-car approach. OPA documentation doesn't really mention this option of deploying multiple OPA services behind a load balancer. However, if you are building your application as a set of Lambda functions, then this might be the way to go:

{{< figure src="/images/posts/open_policy_agent/opa_integration_lambda.png" class="center" >}}

## Utilizing Open Policy Agent APIs

Open Policy Agent comes with a whole set of APIs that you can use in order to utilize OPA to its full potential. I depicted the possible API integrations in the following diagram:

{{< figure src="/images/posts/open_policy_agent/opa_integration_full.png" class="center" >}}

In the diagram above, the green box is your application invoking policy queries against OPA. The purple services are optional services that you can include in your architecture. These services have to be implemented by yourself and they must expose APIs that are specified by OPA. Two blue boxes depict the Prometheus monitoring server and Kubernetes. OPA can integrate with them right away. In the diagram, the direction of the arrows between services is significant. The arrows indicate which service from the pair initiates the TCP connection. In the following subsections, let's take a closer look at each of these OPA interfaces.

###  Open Policy Agent REST API

This is the main API provided by OPA that your application uses to manage data and policies, and to execute queries. It is well described in the OPA's [REST API documentation](https://www.openpolicyagent.org/docs/latest/rest-api/). I would like to highlight two things that I learned about the OPA REST APIs:

First, OPA allows you to set watches on policy queries in order for you to be notified whenever the result of the query evaluation changes. Watches utilize the HTTP long polling mechanism. For additional details, you can refer to the [Watches](https://www.openpolicyagent.org/docs/latest/rest-api/#watches) section of the OPA's documentation.

Second, the communication between the client (i.e. your application) and the OPA service can be protected using TLS. OPA authenticates itself to the client by presenting a valid TLS certificate. Client can authenticate itself to OPA either by presenting a client TLS certificate or by presenting a security token. And guess how OPA handles API authorization? Of course, by evaluating a Rego policy that you supply. For further details on OPA's security settings, refer to the [Security](https://www.openpolicyagent.org/docs/latest/security/) section of OPA's documentation.

### Optional APIs

These APIs are specified by OPA  and you can opt to implement them in order to gain additional functionality and better integrate OPA into your system.

If you recall, in the first part of the series we imported data and policies into OPA by pushing it via the REST APIs. Pushing the data and policies into OPA is not always a feasible option. For example, if OPA is deployed as an ephemeral pod on Kubernetes, it would be difficult to ensure that the API call is made each time right after the pod has started.  How can you make OPA work in such scenarios? You can deploy a service which implements the [Bundle Service API](https://www.openpolicyagent.org/docs/latest/bundles/#bundle-service-api) and configure OPA to periodically pull up-to-date policies and data from this service. In the simplest case, the service can be implemented as a static HTTP server that hosts the policy and data bundles. However, if your use case demands it, you can implement a service that generates policies and data for OPA on-the-fly. When pulling the policies and data, OPA checks the ETag to find out if a new version is available. The Bundle service allows you to ensure that your policies are consistent across many OPA instances and enables you to hot reload them at any time.

OPA can be configured using a set of static configuration files. In Kubernetes, you would likely manage these configuration files as ConfigMaps. However, OPA comes with its own mechanism to manage configuration files from a central place. You can implement the [Discovery Service API](https://www.openpolicyagent.org/docs/latest/discovery/) to expose the configuration files as discovery bundles to OPA instances. On the startup, OPA will download its configuration from the Discovery Service.

OPA can send [status](https://www.openpolicyagent.org/docs/latest/status/) updates to remote HTTP servers that implement a simple [Status Service API](https://www.openpolicyagent.org/docs/latest/status/#status-service-api) . You will be notified whenever OPA downloads and actives a new bundle. Notifications are issued for both policy+data bundles and discovery bundles.

OPA can periodically report [decision logs](https://www.openpolicyagent.org/docs/latest/decision-logs/) to remote HTTP servers that implement a [Decision Log Service API](https://www.openpolicyagent.org/docs/latest/decision-logs/#decision-log-service-api). The reported decision logs record all the policy decisions made by OPA. How could you make use of it? You could implement a simple Decision Log Service that would run co-located with the OPA service and that would store the decision logs into a durable storage like Kafka. And here you go, an awesome audit log was born!

### Health and Monitoring APIs

OPA exposes a [Health API](https://www.openpolicyagent.org/docs/latest/rest-api/#health-api) for you to periodically verify that the OPA service is operational. If you are deploying OPA on top of Kubernetes,  you can leverage the Health API to define the [liveness and readiness probes](https://www.openpolicyagent.org/docs/latest/deployments/#readiness-and-liveness-probes). OPA also exposes Prometheus metrics to [monitor](https://www.openpolicyagent.org/docs/latest/monitoring/) the performance of OPA API calls. Both health and monitoring APIs can be secured the same way as the OPA REST API which we discussed before.

## Where to go from here?

In addition to the written sources that you can find on the web, I would like to point you to a couple of excellent presentations about Open Policy Agent hosted on YouTube:

* [Intro: Open Policy Agent - Torin Sandall, Styra (2018)](https://www.youtube.com/watch?v=CDDsjMOtJ-c)
* [Deep Dive: Open Policy Agent - Torin Sandall & Tim Hinrichs, Styra (2019)](https://www.youtube.com/watch?v=n94_FNhuzy4)

If you are evaluating Open Policy Agent from the security perspective, this [audit report](https://github.com/open-policy-agent/opa/blob/master/SECURITY_AUDIT.pdf) might be of interest to you as well.

## Conclusion

In this article, we discussed several ways for how you can integrate Open Policy Agent with your application. We also described the set of APIs defined by OPA that you can utilize to take full advantage of Open Policy Agent.

Open Policy Agent is a young and fast-moving project. Despite my rather short experience with OPA, I can already recommend that you consider using Open Policy Agent in your project before spending time on implementing your own domain specific language for writing policies or coding your policies in a general-purpose programming language. OPA will keep the policies consistent across your system, will allow you to hot reload the policies at any time and will make policy decisions with very low latency for you.

So far, each of the open source projects came up with its own way how to implement access control. System administrators have to learn how to grant access permissions in the Apache HTTP server, Kubernetes, and fill in your favorite open source project here. It would be great if OPA's Rego language would become an open standard for describing access control rules across the open source ecosystem. I hope that the future will show that this is possible.

Regardless of what other projects decide for themselves, we are planning to utilize Open Policy Agent in the SaaS project I am involved with as a consultant. As we are moving forward,  I intend to share the experience that we gain with OPA with you in some of the future blog posts.

I hope that you found this blog series about Open Policy Agent helpful. If you have any questions or comments, please leave them in the comment section below. I look forward to hearing from you.

