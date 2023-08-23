+++
title = "Evaluating Application Metrics Solutions Prometheus"
date = "2017-09-10"
slug = "2017/09/10/evaluating-application-metrics-solutions-prometheus"
Categories = [ "cloud", "devops", "development" ]
+++

In our [SOA-based](https://en.wikipedia.org/wiki/Service-oriented_architecture) application, the problem of application metrics hasn't been solved yet. We would like to have our application services expose metrics that could be used for monitoring, auto-scaling and analytics. In this blog post, I would like to present to you one of the proposals to solve the application metrics which suggests leveraging [Prometheus](https://prometheus.io/).

<!--more-->

Our application consists of multiple services that are deployed on multiple machines. Currently, our applicaton is deployed on-premise or as a managed offering on the virtual machines in AWS. We're also working on containerizing the application services to achieve higher density and better manageability when deploying into the AWS cloud. Some of our services are so light-weight that we're going to turn them into Lambda functions in the future to further reduce the operational costs. Thus, a solution for application metrics should be able to work in the serverless environment, too.

As of now, we deploy [Icinga](https://www.icinga.com/) along with our application to provide system-level monitoring. Icinga collects the information about the nodes and checks that our services are still running. However, we don't collect any application-level metrics that would allow us to better assess the performance of our system. For example, we would like to know how many requests are processed per second, average request latency, request error rate, what are the depths of the internal queues and so forth. Application metrics would be a welcome input to the auto-scaling decisions and we would like to feed them into our analytics engine as well.

# Getting to know Prometheus

{{< figure src="/images/posts/prometheus_logo.png" height="130" width="130" class="right" >}}

Before jumping in and implementing our own solution for metrics collection and perhaps reinventing the wheel we started shopping around. It seemed to us, that Prometheus monitoring solution was gaining a lot of momentum in recent times. So, we took a closer look at Prometheus and this is what we found:

1.  Prometheus is an open-source monitoring solution hosted by [CNCF](https://www.cncf.io/) - a foundation that hosts Kubernetes as well. Many companies use and contribute to Prometheus.
2.  The [architecture](https://prometheus.io/docs/introduction/overview/) of Prometheus is easy to understand and is modular. While Prometheus provides modules for metrics collection, alerts and Web UI, we would not have to use all of them.
3.  Prometheus is a pull-based monitoring system. Each monitored target has to expose Prometheus formatted metrics. By default, targets make the metrics endpoint available at http://target/metrics. Prometheus periodically scrapes the metrics exposed by the targets.
4.  Our services would need to expose the application metrics in the [Prometheus format](https://prometheus.io/docs/instrumenting/exposition_formats/). There are actually two formats available: a simple text format and protobufs. There are instrumentation [libraries](https://prometheus.io/docs/instrumenting/clientlibs/) for Java, C++ and other languages, to gather the metrics and expose them in the Prometheus format.
5.  The text-based Prometheus metrics format is so simple that it could be collected by other monitoring systems like Nagios or Icinga. Exposing metrics in the Prometheus format doesn't really mandate using Prometheus server for monitoring.
6.  There's a Prometheus [jmx_exporter](https://github.com/prometheus/jmx_exporter) library to convert the JMX MBeans data into Prometheus format. This would come in handy for gathering Tomcat metrics, for example.
7.  [Dropwizard metrics](http://metrics.dropwizard.io/) is a popular Java instrumentation library. For instance, Vert.x toolkit can report its [internal metrics](http://vertx.io/docs/vertx-dropwizard-metrics/java/) using the Dropwizard metrics library and there are other frameworks that supports it. Prometheus comes with a [simpleclient_dropwizard](https://github.com/prometheus/client_java/tree/master/simpleclient_dropwizard) library that can make Dropwizard metrics available to Prometheus monitoring.
8.  To prevent unauthorized access, the metric targets would need to be protected using TLS in combination with client certs, bearer token or HTTP basic authentication.
9.  Prometheus pulls the metrics from the monitored targets. In addition, Prometheus comes with a [Pushgateway](https://prometheus.io/docs/practices/pushing/) where clients can push their metrics to. However, as noted in the Prometheus documentation: *Usually, the only valid use case for the Pushgateway is for capturing the outcome of a service-level batch job*. Hence, Pushgateway would not work for aggregating metrics pushed by the Lambda functions.
10.  In addition to application-level metrics, system-level metrics can be collected by Prometheus as well thanks to the [node_exporter](https://github.com/prometheus/node_exporter).
11.  Prometheus is a great fit for dynamic environments like clouds and container clusters due to its discovery capabilities. In AWS, operator attaches tags to VMs and based on that Prometheus can discover them and start monitoring them automatically. The same principle works for container clusters like Kubernetes, too. One has to add annotations to pods and Prometheus will discover them automatically.
12.  Prometheus makes the collected metrics available for querying via an [HTTP API](https://prometheus.io/docs/querying/api/). We could retrieve the metrics using this API in order to feed them into our analytics engine.
13.  There is a great [guide](https://prometheus.io/docs/practices/naming/) that would help us when designing our custom metrics.
14.  Prometheus is written in Go and comes in a form of statically-linked binaries. This makes the installation of Prometheus a breeze.

# Instrumenting Java applications

In order to gather application metrics and to make them available to the Prometheus monitoring system, we would need to instrument our application services using Prometheus libraries. To get a clear idea, we created a proof-of-concept Java application instrumented using Prometheus. You can find it on [GitHub](https://github.com/noseka1/prometheus-poc).

Alternatively, we are thinking about leveraging Dropwizard metrics library for instrumentation. The Dropwizard metrics library is rather popular and is not connected with any particular monitoring solution. We would still be able to expose the Dropwizard metrics to Prometheus using a wrapper [simpleclient_dropwizard](https://github.com/prometheus/client_java/tree/master/simpleclient_dropwizard).

# Monitoring AWS Lambda functions

AWS Lambda functions are extremely short-lived processes. Prometheus won't be able to pull the application metrics from them. Instead, Lambdas will have to push their metrics to Prometheus. At the first glance, we thought that the Prometheus Pushgateway could help here, however, reading the Pushgateway's documentation more carefully we found that *the Pushgateway is explicitly not an aggregator or distributed counter but rather a metrics cache*. And that's a problem, as we would like to count how many Lambda instances are being invoked per second and so on.

At the moment, we can see two approaches how to make the monitoring of Lambda functions work with Prometheus. Either, push the application metrics from the Lambda functions using a StatsD client. Prometheus' [statsd_exporter](https://github.com/prometheus/statsd_exporter) would play a role of a StastD server and make the metrics available to Prometheus. Or, the second approach would be to create our own metrics aggregator that would receive the metrics from Lambda functions in the Prometheus format, aggregate them and expose them to the Prometheus server.

# Alternatives

Besides using Prometheus, we were also thinking about other solutions for application metrics. As we already deploy Icinga for the system-level monitoring, it would make sense to use it for application metrics, too. We really like Icinga, it's a great monitoring software. Unfortunately, Icinga is based on the node and services model where a statically configured set of nodes are running services on them. This doesn't really fit with the modern containerized deployments where containers are dynamically scheduled on the cluster nodes and are also scaled up and down. Also, Prometheus server supports all sorts of metric queries and aggregations. Icinga is lacking this feature altogether. That's why we're leaning towards replacing Icinga with Prometheus for system-level as well as application-level monitoring.

[Hawkular](http://www.hawkular.org/) seems to be another modern monitoring project we would like to take a closer look at. In contrast to Prometheus project which is developed by many parties, it seems that Hawkular project is mostly driven by Red Hat.

# Conclusion

Prometheus is a modern monitoring system. It was the first system we evaluated as we were trying to find a good solution for application metrics. In addition to application-level metrics, we could use Prometheus to collect system-level metrics as well. This would make Prometheus a single monitoring solution for our application. The only bigger issue for us is the absence of the AWS Lambda monitoring story.

If you have an application that you deliver on-premise as well as in the cloud, how did you solve the application metrics collection and monitoring? Is Prometheus a good way to go? Please, leave your comments below.

