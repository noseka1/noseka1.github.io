+++
title = "Edge Security for Your Cloud Application Part I"
date = "2018-01-10"
slug = "2018/01/10/edge-security-for-your-cloud-application-part-i"
Categories = []
+++

When designing a cloud application, one of the challenges you need to tackle is the secure communication of your clients with your servers over the public Internet. In this post, I'm going to sketch a system architecture that allows you to authenticate the clients and exchange data between your clients and servers securely. The proposed architecture is also suitable when migrating an existing application, which didn't implement the secure communication, to the cloud.

<!-- more -->

In this article, we are going to assume that we're dealing with a multi-tenant cloud application whose architecture follows the [SOA](https://en.wikipedia.org/wiki/Service-oriented_architecture) architecture principles. The application consists of several edge services that accept the incoming client requests. After validating the incoming request, the request is passed to the backend services for processing. Here is a diagram depicting such an application:

{% img /images/posts/edge_security_for_your_cloud_application_soa.svg 800 1000 SOA architecture %}

## Requirements

Before we start walking through the design, let's take a look at the set of requirements that drove the design decisions:

1. One of the first steps of request validation should be the client authentication. Requests originating from the clients that fail to authenticate should be refused immediately.

2. We don't impose any restrictions on what protocol is used by the clients to communicate with our application. Typically, clients will leverage the HTTP protocol but we want to support any TCP based protocol. In other words, our application doesn't have to be a web service.

3. From the past, we inherited some clients that are not able to communicate in a secure way. For instance, they are only able to talk plain HTTP and don't support HTTPS. We would like to make such clients work with our application without the need to modify the clients themselves.

4. And last but not least, we would like to avoid the need to establish a VPN connection between the client machine and the cloud. In practice, setting up a VPN connection requires quite invasive configuration of the client operating system. For our application tenants (our customers) this would impose a barrier that would hurt the adoption of our cloud application. Something we definitely don't want.

## The edge layer

SSL/TLS is a family of security protocols that many VPNs rely upon. Currently, the TLS protocol is considered to be one of the strongest and most mature security protocols available. It was a clear choice for us to leverage the TLS protocol for communication between the clients and servers including the mutual authentication using PKI certificates. Finally, here is a diagram showing the edge layer of our cloud application in detail:

{% img /images/posts/edge_security_for_your_cloud_application_server.svg 800 1000 The edge layer %}


## Internet-facing ELB

Interestingly, AWS Elastic load balancers don't support client certificate authentication. Both, Classic load balancer and Application load balancer, support TLS offloading where they can terminate the TLS connection for you. However, they are not able to authenticate the client certificate and they also don't allow forwarding the certificate details to the backend application which could carry out the authentication by itself. In order to implement the client certificate authentication, you are pretty much left with two options: you can either use the Classic load balancer in the TCP mode, or you can employ the Network load balancer which operates on the TCP level. In both cases, the Elastic load balancer just passes the TLS connection through to your EC2 instances where you have to terminate and authenticate the TLS connection yourself. As the connection between ELB and the EC2 instances remains encrypted, you are gaining a plus from the security standpoint, too.

While ELB doesn't authenticate clients, it plays an important role in our architecture. First, ELB is highly-available and distributes the traffic to several edge instances that are not highly-available. Second, ELB implements a layer 3 (e.g. UDP reflection) and layer 4 (e.g. SYN flood) DDOS attack protection.

Btw., Amazon API Gateway doesn't support client certificate authentication either and so is less helpful for our scenario.

## Reverse proxy on edge instances

Depending on your architecture, your edge services may or may not have a support for accepting TLS connections built in. I personally see the connection security to be a job for the infrastructure and would vote against implementing the TLS support in your application services. Here are some concrete arguments why to terminate the TLS connection outside of your application services:

1. Your application services may be written in different languages. For each language and its runtime, the TLS configuration is different. You don't want to spend your time figuring out, how to configure TLS on Apache Tomcat, Eclipse Jetty, Apache server, Node.js and others, do you? Different technologies support different TLS features. For example, TLS SNI is supported only in Tomcat >= 8.5. Researching the supported feature set of every runtime is time consuming.

2. The proxy provides a good place to monitor and log what's going on on the wire. Remember that ELB won't have any insight into your traffic as the traffic is TLS encrypted.

3. You may want to automate the TLS certificate rotation and revocation. You'll have to implement this for all your different web servers. That's quite a bit of work.

Instead of handling TLS in your application services, I would recommend deploying a reverse proxy in front of your services. This proxy will terminate the incoming TLS connection. This will allow you to manage all the TLS related settings in one place. The battle-tested proxies like [HAProxy](http://www.haproxy.org/) or [NGINX](https://www.nginx.com/) can authenticate the client certificate against a trusted certificate authority. They can also forward the certificate details to your edge service which can implement a more complex authentication logic if you need it.

You should deploy the reverse proxy on the same instance with your edge service. This way, the decrypted communication between the proxy and your service will never leave the instance. In order to capture the decrypted traffic, one would need to have a root access on the instance. The possibility to capture the communication in the clear will come handy when troubleshooting, though.

## The client side

After discussing the server-side design, let's talk about the client-side part of the picture. As we mentioned in the requirements section above, our architecture has to also support clients that don't have the TLS functionality built in. In turns out that there are actually three different client types:

1. Clients with native TLS mutual authentication support
2. Clients with no TLS support at all
3. Clients supporting TLS with one-way authentication only, i.e. the client is able to authenticate the server certificate but it doesn't present its own certificate to the server.

There is nothing special to do for the clients with native TLS mutual authentication support. They can directly connect to the application servers. To ensure the secure communication for the remaining two client types, we're going to deploy a proxy on the client machines. This proxy will enforce the TLS mutually authenticated connection over the Internet. A diagram depicting the three client types looks as follows:

{% img /images/posts/edge_security_for_your_cloud_application_client.svg 600 800 The client side %}

In the case number two, the proxy is configured in the SSL/TLS encryption mode. It accepts an unencrypted connection from the client and forwards the communication over a TLS encrypted channel to the server.  For the case number three, the proxy must be configured using the SSL/TLS bridging aka re-encryption mode. The proxy accepts an encrypted connection from the client, creates a separate encrypted connection to the server and passes the data between the two connections. Both HAProxy and Nginx support these configuration modes. You can also refer to the TLS layouts described in the HAProxy [documentation](https://www.haproxy.com/documentation/aloha/7-0/deployment-guides/tls-layouts/).

Alternatively, as a proxy one could also employ [stunnel](https://www.stunnel.org). Before the SSL/TLS support was built into HAProxy, stunnel used to be deployed along with HAProxy to provide the SSL/TLS functionality. To accomplish the case number three using stunnel, one would actually need to combine two stunnels in series.

Recently, a modern [Envoy](https://www.envoyproxy.io/) proxy emerged and I would like to encourage you to check it out. It provides a really impressive set of features that goes far beyond the load balancing functionality: automatic retries, circuit breaking, zone local load balancing, very detailed metrics etc. Envoy helps to solve several networking problems common to the cloud-native applications. In our proposed architecture, Envoy would be deployed in the role of an edge proxy on the server side as well as in the role of the service proxy on the client side.

Whichever proxy you choose on the client-side, remember to deploy it in a highly available fashion. You don't want to introduce a single point of failure into your system, do you?

## Conclusion

In this post, we walked through the secure edge design explaining the reasoning behind the individual design decisions. In the [second blog post](/blog/2018/01/12/edge-security-for-your-cloud-application-part-ii) of this miniseries, we're going to demonstrate a practical implementation of our approach using HAProxy.
