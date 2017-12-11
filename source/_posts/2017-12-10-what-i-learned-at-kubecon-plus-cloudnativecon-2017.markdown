---
layout: post
title: "What I Learned at KubeCon + CloudNativeCon 2017"
date: 2017-12-10 16:16:25 -0800
comments: true
categories:  events
---

KubeCon + CloudNativeCon North America 2017 took place in Austin, Texas on December 6-8, 2017. I greatly enjoyed this conference. This blog post presents some of the notes I made during the conference talks. Would you like to learn about Amazon EKS, Kata Containers or the CRI-O project? Do you want to better understand Docker's Moby project and its relation to LinuxKit and InfraKit? Then read on!

<!-- more -->

{% img center /images/posts/kubecon_2017.png 500 200 %}

## Keynotes

* **Keynote: A Community of Builders**
  * Linux Foundation created a [Introduction to Kubernetes](https://www.edx.org/course/introduction-kubernetes-linuxfoundationx-lfs158x) course available to everybody who wants to learn Kubernetes. The course is provided free of charge on the edX platform.
  * CNCF in collaboration with The Linux Foundation created a certification program [Certified Kubernetes Administrator](https://www.cncf.io/certification/expert/) (CKA). Check it out, if you want to demonstrate your Kubernetes skills.
* **Keynote: CNCF Project Updates**
  *  The recently released [CoreDNS](https://coredns.io/) v1.0 is going to replace [Kube DNS](https://github.com/kubernetes/dns) in Kubernetes v1.9.
  * Container runtime [containerd](https://containerd.io/), originally created by Docker and now hosted by CNCF, reached its v1.0 milestone.
  * Today, the CNCF logging project [Fluentd](https://www.fluentd.org/) reached its v1.0 release.
  * A rather young project [Fluent Bit](http://fluentbit.io/) is a log forwarder which allows you to collect your data/logs from various sources and forward them to multiple destinations. Supported destinations are among others: Elasticsearch, InfluxDB or Kafka. Fluent Bit is implemented in C and it only leverages asynchronous operations to collect and deliver data. It integrates well with the [Prometheus](https://prometheus.io/) monitoring system.
* **Keynote: Cloud Native at AWS**
  * AWS chose containerd as the runtime for their managed Kubernetes service [EKS](https://aws.amazon.com/eks/).
  * With the recent announcements of [Amazon EC2 Bare Metal instances](https://aws.amazon.com/about-aws/whats-new/2017/11/announcing-amazon-ec2-bare-metal-instances-preview/) and [Amazon Elastic Container Service for Kubernetes](https://aws.amazon.com/blogs/aws/amazon-elastic-container-service-for-kubernetes/), there are now four first-class deployment entities available on AWS: bare metal, VMs, containers and functions.
  * Amazon ECS for Kubernetes will leverage IAM to authenticate identities and the Kubernetes' RBAC for authorization. Pods will be assigned IAM roles in order for them to be able to talk to other AWS services, e.g. S3.

## Sessions attended

* **Embedding the Containerd Runtime for Fun and Profit**
  * ShiftFS is trying to solve the problem with the access (uid/gid) to a single file system from two containers running in different user namespaces.
* **Kata Containers: Hypervisor-Based Container Runtime**
  * The [Kata Containers](https://katacontainers.io/) project is a merger of two projects: [Intel Clear Containers](https://github.com/clearcontainers/runtime) and [Hyper runV](https://github.com/hyperhq/runv). The goal of the project is to create a container runtime that would run containers as virtual machines, however, with the speed comparable to the namespace-based containers. This would allow to bring the secure hypervisor-based isolation to containers.
  * Kata Containers would provide an alternative to the [runc](https://github.com/opencontainers/runc) runtime which is used by the namespace-based containers. In the Kubernetes cluster, both namespace-based as well as hypervisor-based containers could run at the same time.
  * On Kubernetes, a Kubernetes pod would be implemented as a single VM.
* **Building Specialized Container-Based Systems with Moby: A Few Use Cases**
  * [Moby project](https://blog.docker.com/2017/04/introducing-the-moby-project/) is pretty much an umbrella for all Docker open-source projects. Projects like [containerd](https://github.com/containerd/containerd), [LinuxKit](https://github.com/linuxkit/linuxkit), [InfraKit](https://github.com/docker/infrakit), and many more are part of the Moby project. From the components of the Moby project, Docker company builds their Docker SE product, from which the Docker EE product is derived. Moby project is governed by the Technical Steering Committee ([TSC](https://github.com/moby/tsc)). In times before the TSC was established, technical disputes were resolved by the Benevolent Dictator for Life (BDFL) which was Solomon Hykes.
  * [LinuxKit](https://github.com/linuxkit/linuxkit) is a toolkit for building custom Linux images. The single purpose of these images is to run containerized applications. When creating an image, you can choose a Linux kernel, choose a container runtime and pick additional services that should be started on boot. If the basic containerd runtime doesn't suit you, LinuxKit can even install Kubernetes for you. Resulting images include only packages that you've chosen and nothing else, hence reducing the attack surface and improving security. You can spin AWS instances off of your customized images and start scheduling containerized applications on them. Instances are considered immutable. In order to apply software updates, you must build a new version of the image and replace the instances running the old image.
  * In order simplify the managing of the infrastructure running on top of immutable images (built with LinuxKit), Docker came up with another toolkit called [InfraKit](https://github.com/docker/infrakit.git). InfraKit allows you to create and manage clustered infrastructure like Swarm or Kubernetes clusters. Docker's motivation behind InfraKit was the deployment and management of their Docker SE and Docker EE products. Behind the scenes, InfraKit leverages Terraform for resource provisioning.
* **CRI-O: All the Runtime Kubernetes Needs and Nothing More**
  * Container runtime containerd originally developed by Docker Inc. is used for various purposes. Besides Kubernetes and OpenShift, containerd powers Swarm clusters and the recently announced Amazon EKS is also going to leverage containerd. Being a general purpose runtime, containerd suffered from compatibility issues with Kubernetes, where an update of containerd would break the Kubernetes cluster. For the sake of stability and [several other reasons](http://www.projectatomic.io/blog/2017/06/6-reasons-why-cri-o-is-the-best-runtime-for-kubernetes/), Red Hat initiated a project [CRI-O](http://cri-o.io/) which aims to create an alternative runtime that would be dedicated to Kubernetes only. CRI-O is available for testing in OpenShift 3.7 and it is going to ship as the default container runtime in the future OpenShift releases, replacing containerd.
* **Extending Kubernetes 101**
  * Kubernetes can be extended by implementing a custom controller also called *operator*.
  * This controller runs as a pod on Kubernetes cluster. On startup, custom controller is supposed to register one or more custom resources with Kubernetes. After that, the controller watches the changes in the desired state of custom resources and acts upon them.
  * The concept of [Custom Resources](https://kubernetes.io/docs/concepts/api-extension/custom-resources/) is supported by Kubernetes since version 1.7.
  * Custom resources are designed to feel like native Kubernetes resources. They are exposed as another API along with the native APIs. The kubectl client can discover and work with custom resources right away.
  * Using the kubectl, user can change the desired state of custom resources (add, update, delete). The custom controller that watches the desired state of the custom resources will carry on operations required to reach the desired state.
  * Btw., there's a generator available that speeds up the implementation of operators.
  * See also [Kubernetes Operator Kit](https://github.com/rook/operator-kit).
* **Certifik8ts: All You Need to Know about Certificates in Kubernetes**
  * As of Kubernetes 1.8, the Kubelet can request a new client certificate when the current one is nearing its expiration.
  * Currently, Kubernetes cannot revoke certificates. This is good to know if you use client certificates to athenticate users. Once you issue a client certificate to the user, you won't be able to revoke it.
* **Istio: Saling to a Secure Services Mesh**
  * Service mesh acts as a proxy on the communication path between your microservices. As a proxy, service mesh adds a host of useful functionality that you would otherwise have to implement yourself in your microservices: secure communication, traffic routing, load balancing, rate limiting, circuit breakers, timeouts and retries, metrics, reporting, etc.
  * Somebody on the conference has noted, that the year 2018 will be the year of the service mesh. Based on how many problems the service mesh can solve for distributed microservices, I can only agree with that.
  * In this talk, the security features of [Istio](https://istio.io/) service mesh were presented.
  * Istio can establish a mutually authenticated TLS connections between your microservices, without the need to change your application code. It solves the problem of bootstrapping the TLS trust for you by establishing a custom CA.
  * Istio Ingress can expose the microservice outside of the service mesh.
  * Istio Egress controls which external hosts (hosts outside of the service mesh) your microservices can talk to.
* **Cost-effective Compute Clusters with Spot and Pre-emptible Instances**
  * Quote: *Kubernetes is going to make spot instances mainstream*.
  * What applications work well with spot instances? Elastic/bursting applications, HPC workloads, HA cluster apps and horizontally scalable apps.
