---
layout: post
title: "What I Learned at Red Hat Summit 2017"
date: 2017-05-05 18:46:14 -0700
comments: true
categories: events
---

I had the great opportunity to visit the Red Hat Summit 2017. It was hosted in the Boston Convention and Exhibition Center in Boston in May 2-4, 2017. This blog post summarizes the interesting things I learned at the summit.

<!-- more -->

## Major announcements

{% img right /images/posts/rh_summit.png 200 200 %}

**[Red Hat OpenShift & Amazon Web Services](https://www.redhat.com/en/about/press-releases/red-hat-and-aws-extend-strategic-alliance-package-access-aws-services-within-red-hat-openshift).** Red Hat will make AWS services accessible directly from the OpenShift web console. From within the OpenShift web console developers will be able to provision and configure AWS services such as CloudFront, ElastiCache, ELB, RDS, EMR, RedShift, S3 and Lambda.

**[Red Hat OpenShift.io](https://www.redhat.com/en/about/press-releases/red-hat-unveils-end-end-cloud-native-development-environment-red-hat-openshiftio).** [OpenShift.io](https://openshift.io/) is an online development environment for building container-based applications. OpenShift.io can create a project in GitHub for you to store your source code. The source code editing can be done in the integrated Eclipse Che. OpenShift.io can create a project in OpenShift Online in order to build your application and deploy it. Jenkins pipelines are used to orchestrate the CI/CD process. Currently, OpenShift.io is available in a limited developer preview. You can sign up at https://openshift.io

## Sessions attended
- **The future Red Hat Middleware portfolio: Stacks and services and solutions**
  - Netflix OSS is interesting and Red Hat will support it because customers like it. However, some of the Netflix OSS features might be more efficiently implemented directly in Kubernetes instead of on top of Kubernetes. Netflix had to make architectural choices based on how AWS looked several years ago. AWS evolved since that time.
  - Architectural evolution: monolith -> n-tier architecture -> microservices.
- **Red Hat container technology strategy**
  - Kubernetes has won. You may have just not realized it yet.
  - There are two reasons why the Kubernetes open source project is winning. First, it brings value to people. Second, people are excited to work on it.
  - Kubernetes = kernel of the cloud operating systems
- **Reproducible development to live applications with Java and Red Hat CDK**
  - When developing containerized applications, developers can run them locally (outside of a container), locally on [MiniShift](https://github.com/minishift/minishift), or hosted on [OpenShift Online](https://www.openshift.com/).
  - [Red Hat Container Development Kit](https://developers.redhat.com/products/cdk) can help you develop container-based applications quickly. It uses MiniShift under the hood.
- **Modern Java and DevOps lightning talks**
  - You can issue `oc cluster up` to create a local OpenShift all-in-one cluster (requires Origin >= 1.3).
  - [Eclipse MicroProfile](https://projects.eclipse.org/proposals/eclipse-microprofile) project is aimed at optimizing Enterprise Java for the microservices architecture. It focuses, among others, on application configuration, health-checking, fault tolerance and security.
  - You can use API Gateway (e.g [Apigee](https://apigee.com/about/cp/api-gateway), [3scale](https://www.3scale.net/)) to dynamically route traffic into different OpenShift namespaces (test, staging, production).
- **Atomic BOF**
  - In the future, the classic RHEL will be derived from the RHEL Atomic Host. It means that the new features will appear in the RHEL Atomic Host before being included into RHEL.
- **Stepping off a cliff: Common sense approaches to cloud security**
  - VMs in the cloud are created and destroyed dynamically. This is one of the challenges for the security team.
- **Wicked fast PaaS: Performance tuning of OpenShift and Docker**
  - RHEL 7.4 should support OverlayFS. For Docker storage, the OverlayFS is more memory efficient than the LVM thin pool provisioning, as with OverlayFS, the pages in the page cache can be shared between multiple containers.
  - Beginning with OpenShift 3.5, the container image metadata will be stored only in the Docker registry. In previous versions of OpenShift, a duplicate of the image metadata was stored in etcd, too.
- **The Truth about Microservices**
  - "Building a single microservice is easy. Building a microservices architecture is hard."
- **The hardest part of microservices is your data**
  - [Debezium](http://debezium.io/) monitors the changes committed to the database (MySQL, MongoDB, PostgreSQL). For each database change, Debezium publishes an event to the Kafka broker. To consume the change events, an application can create a Kafka consumer that will consume all events for the topics associated with the database. In summary, Debezium turns a database transaction log into a Kafka stream that other applications can consume.
- **Reactive systems with Eclipse Vert.x and Red Hat OpenShift**
  - [Vert.x](http://vertx.io/) is a toolkit for building reactive applications on the JVM. It can discover services on OpenShift and Kubernetes.
- **Container infrastructure trends: Optimizing for production workloads**
  - [skopeo](https://github.com/projectatomic/skopeo) is a command line utility that allows you to inspect Docker images and image registries. It retrieves the required information from the repository metadata without the need to download the actual image. For example, with skopeo you can find out which tags are available for the given repository.
  - [buildah](https://github.com/projectatomic/buildah) a tool for building Docker images. It can build container images without using the Docker daemon. Ansible-container and OpenShift's S2I will be modified to use buildah under the hood.
- **Function as a Service (Faas) - why you should care and what you need to know**
  - Architectural evolution: service -> microservice -> function.
  - Good serverless use-cases: processing web-hooks, scheduled tasks (a la cron), data transformation (converting small images).
  - Serverless architecture challenges: cannot use a larger programming framework to implement the function, as this would be too slow to initialize, increased latency in comparison to a long-running server process, large variance in latency, very complex at scale, debugging of functions is hard.
  - Red Hat participates on development of [Funktion](https://funktion.fabric8.io/) that is part of the project [fabric8](https://fabric8.io/).
