+++
title = "Configuring Envoy to Audo Discover Pods on Kubernetes"
date = "2019-08-19"
slug = "2019/08/19/configuring-envoy-to-audo-discover-pods-on-kubernetes"
Categories = [ "cloud", "devops" ]
+++

Pods on Kubernetes are ephemeral and can be created and destroyed at any time. In order for Envoy to load balance the traffic across pods, Envoy needs to be able to track the IP addresses of the pods over time. In this blog post, I am going to show you how to leverage Envoy's Strict DNS discovery in combination with a headless service in Kubernetes to accomplish this.

<!--more-->

## Overview

Envoy provides several [options](https://www.envoyproxy.io/docs/envoy/v1.10.0/intro/arch_overview/service_discovery) on how to discover back-end servers. When using the [Strict DNS](https://www.envoyproxy.io/docs/envoy/v1.10.0/intro/arch_overview/service_discovery#strict-dns) option,  Envoy will periodically query a specified DNS name. If there are multiple IP addresses included in the response to Envoy's query, each returned IP address will be considered a back-end server. Envoy will load balance the inbound traffic across all of them.

How to configure a DNS server to return multiple IP addresses to Envoy? Kubernetes comes with a [Service](https://kubernetes.io/docs/concepts/services-networking/service/) object which, roughly speaking, provides two functions. It can create a single DNS name for a group of pods for discovery and it can load balance the traffic across those pods. We are not interested in the load balancing feature as we aim to use Envoy for that. However, we can make a good use of the discovery mechanism. The Service configuration we are looking for is called a [headless service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) with selectors.

The diagram below depicts how to configure Envoy to auto-discover pods on Kubernetes. We are combining Envoy's Strict DNS service discovery with a headless service in Kubernetes:

{{< figure src="/images/posts/envoy_auto_discovery.png" class="center" >}}

## Practical implementation

To put this configuration into practice, I used [Minishift](https://www.okd.io/minishift/) 3.11 which is a variant of Minikube developed by Red Hat. First, I deployed two replicas of the httpd server on Kubernetes to play the role of back-end services. Next, I created a headless service using the following definition:

{{< highlight yaml "linenos=table" >}}
apiVersion: v1
kind: Service
metadata:
  name: httpd-discovery
spec:
  clusterIP: None
  ports:
    - name: http
      port: 8080
  selector:
    app: httpd
  type: ClusterIP
{{< / highlight >}}

Note that we are explicitly specifying "None" for the cluster IP in the service definition. As a result, Kubernetes creates the respective Endpoints object containing the IP addresses of the discovered httpd pods:

{{< highlight shell "linenos=table" >}}
$ oc get endpoints
NAME              ENDPOINTS                                                        AGE
httpd-discovery   172.17.0.21:8080,172.17.0.22:8080                                30s
{{< / highlight >}}

 If you ssh to one of the cluster nodes or rsh to any of the pods running on the cluster, you can verify that the DNS discovery is working:

{{< highlight shell "linenos=table" >}}
$ host httpd-discovery
httpd-discovery.mynamespace.svc.cluster.local has address 172.17.0.21
httpd-discovery.mynamespace.svc.cluster.local has address 172.17.0.22
{{< / highlight >}}

Next, I used the container image `docker.io/envoyproxy/envoy:v1.7.0` to create an Envoy proxy. I deployed the proxy into the same Kubernetes namespace called `mynamespace` where I created the headless service before. A minimum Envoy configuration that can accomplish our goal looks as follows:

{{< highlight yaml "linenos=table" >}}
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address:
        protocol: TCP
        address: 0.0.0.0
        port_value: 10000
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        config:
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  host_rewrite: httpd
                  cluster: httpd
          http_filters:
          - name: envoy.router
  clusters:
  - name: httpd
    connect_timeout: 0.25s
    type: STRICT_DNS
    dns_lookup_family: V4_ONLY
    lb_policy: ROUND_ROBIN
    hosts:
      - socket_address:
          address: httpd-discovery
          port_value: 8080
admin:
  access_log_path: /tmp/admin_access.log
  address:
    socket_address:
      protocol: TCP
      address: 127.0.0.1
      port_value: 9901
{{< / highlight >}}

Note that in the above configuration,  I instructed Envoy to use the Strict DNS discovery and pointed it to the DNS name `httpd-discovery` that is managed by Kubernetes.

That's all that was needed to be done! Envoy is load balancing the inbound traffic across the two httpd pods now. And if you create a third pod replica, Envoy is going to route the traffic to this replica as well.

## Conclusion

In this article, I shared with you the idea of using Envoy's Strict DNS service discovery in combination with the headless service in Kubernetes to allow Envoy to auto-discover the back-end pods. While writing this article, I discovered this [blog post](https://blog.markvincze.com/how-to-use-envoy-as-a-load-balancer-in-kubernetes/) by Mark Vincze that describes the same idea and you should take a look at it as well.

This idea opens the door for you to utilize the advanced features of Envoy proxy in your microservices architecture. However, if you find yourself looking for a more complex solution down the road, I would suggest that you evaluate the [Istio](https://istio.io/) project. Istio provides a control plane that can manage Envoy proxies for you achieving the so called service mesh.

Hope you found this article useful. If you are using Envoy proxy on top of Kubernetes I would be happy to hear about your experiences. You can leave your comments in the comment section below.
