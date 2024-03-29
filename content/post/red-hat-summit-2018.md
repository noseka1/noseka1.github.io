+++
title = "Red Hat Summit 2018"
date = "2018-05-14"
slug = "2018/05/14/red-hat-summit-2018"
Categories = [ "events" ]
+++

Last week, I had the pleasure of attending Red Hat Summit 2018. It was hosted at the Moscone Center in San Francisco, May 8-10, 2018. I greatly enjoyed this conference and would like to share with you some of the interesting things I learned there.

<!--more-->

You can view video recordings of the general sessions as well as a selection of the breakout sessions on [YouTube](https://www.youtube.com/user/redhatsummit).

# Major announcements

[Red Hat OpenShift on Azure](https://www.redhat.com/en/about/press-releases/red-hat-and-microsoft-co-develop-first-red-hat-openshift-jointly-managed-service-public-cloud). Red Hat and Microsoft are introducing OpenShift as a fully managed service on Azure. This brings customers the possibility to freely move workloads between their on-premise OpenShift clusters and Azure. Many of the Azure services will be accessible from within OpenShift via the Service Catalog. In the future, Windows containers will be supported alongside Linux containers. Support for Red Hat OpenShift on Azure will be provided by both Red Hat and Microsoft.

[Red Hat and IBM working toward a hybrid cloud](https://www.redhat.com/en/about/press-releases/ibm-and-red-hat-join-forces-accelerate-hybrid-cloud-adoption). Red Hat and IBM are expanding their collaboration to bring together enterprise application platforms, Red Hat OpenShift and IBM Cloud Private. IBM's recent move to re-engineer its entire software portfolio with containers, including WebSphere, MQ Series and Db2 will allow it to make this sofware available on the hybrid cloud with IBM Cloud Private and Red Hat OpenShift serving as the common foundation.

Both major announcements go along with the Red Hat's vision of the future being the hybrid cloud. There's no doubt that Red Hat stands behind the hybrid cloud. In one of the general sessions, I heard Red Hat's Paul Cormier saying:

> Hybrid cloud is the only practical way forward.

# Sessions attended

- **CoreOS and Red Hat**
  - After Red Hat [acquired](https://www.redhat.com/en/about/press-releases/red-hat-acquire-coreos-expanding-its-kubernetes-and-containers-leadership) CoreOS in January this year, developers from both companies are busy with merging features of CoreOS Tectonic into Red Hat OpenShift. While CoreOS Tectonic was focused on operators, Red Hat OpenShift was more oriented toward developers. After the merge is completed, Red Hat OpenShift should become a product with a strong appeal to both operators and developers.
  - OpenShift console will be extended by an Admin console which is based on the Tectonic Console.
  - OpenShift is embracing the [Operator Framework](https://github.com/operator-framework) originally developed by CoreOS. Initially, more than 60 ISVs are planning to provide certified OpenShift operators to end users.
  - Quay Enterprise is going to be rebranded to Red Hat Quay. It remains a standalone product. Customers with higher demands may prefer leveraging Red Hat Quay in place of the integrated OpenShift image registry.
  - CoreOS Container Linux and RHEL Atomic Host are going to be merged into a new container operating system called Red Hat CoreOS. Initially, Red Hat CoreOS is going to be based on RHEL 7.5, however, later on Red Hat CoreOS will be updated faster than RHEL.
- **Next-generation tools for container technology**
  - Dan Walsh discussed several Linux container related tools developed by his team: [CRI-O](https://github.com/kubernetes-incubator/cri-o) container runtime, [Buildah](https://github.com/projectatomic/buildah), [Skopeo](https://github.com/projectatomic/skopeo/), and [Podman](https://github.com/projectatomic/libpod). With a certain dose of sarcasm, Dan explained the flaws in the design of the Docker tool -  mainly the unfortunate decision to create a "big fat" Docker daemon. Podman is a command-line tool that reimplements the Docker functionality without the need for a Docker daemon. In the future, Podman could replace the Docker tool.
- **Kubernetes and the platform of the future**
  - This was a chat with Clayton Coleman (Chief Engineer for OpenShift) and Brandon Philips (previously CTO of CoreOS, acquired by Red Hat) about the future of Kubernetes and OpenShift projects.
  - Kubernetes is considered to be a new distributed operating system. "We have a deployment target for the open-source community for multiple machines, as we had Linux as a target for a single machine".
- **Low-risk mono to microservices: Istio, Teiid, and Spring Boot**
  - In this session, I learned about several open-source projects that may come in handy when moving toward a microservice architecture: [Arquillian](http://arquillian.org/) is an integration testing framework, [Istio](https://istio.io/) is a service mesh, [Kiali](https://github.com/kiali/kiali) is an Istio observability project, [apicurio](https://www.apicur.io/) is a tool for designing RESTful APIs according to the [OpenAPI](https://github.com/OAI/OpenAPI-Specification) specification, [Teiid](http://teiid.jboss.org/) is a data virtualization system.
  - If you want to release your software frequently, monolithic architecture may become a bottleneck. You can begin to break up your monolithic project into microservices to speed up the release process. If you've gotten to the point where you can go faster, stop breaking things up! You don't want to cross the point of diminishing returns. If there is some monolith left in your project, that's okay.
- **OpenShift roadmap: You won't believe what's next**
  - Prometheus monitoring system is going to be integrated into OpenShift more deeply. In addition to monitoring, metrics collected by Prometheus are going to be used for container auto healing.
  - Work is underway for Open Virtual Network (OVN) to become the default OpenShift SDN in the future.
  - Red Hat is investing in the [KubeVirt](https://github.com/kubevirt) project whose goal is to run virtual machines on Kubernetes/OpenShift.
  - Red Hat is also investing in [Apache OpenWhisk](https://openwhisk.apache.org/). OpenWhisk is a serverless platform allowing to run functions on Kubernetes.
- **Intelligent applications on OpenShift from prototype to production**
  - OpenShift is entering the big data arena. This was an introductory talk about running Spark on OpenShift for machine learning. If you are interested to run data analysis on OpenShift, check out the [rad analytics](https://radanalytics.io/) project.

# Some photos from the event

{{< figure src="/images/posts/redhatsummit2018/keynote.jpeg" class="center" >}}

Opening keynote session.

{{< figure src="/images/posts/redhatsummit2018/cruise.jpeg" class="center" >}}

Harbor cruise for Red Hat Certified Professionals, including myself ;-)

{{< figure src="/images/posts/redhatsummit2018/sf1.jpeg" class="center" >}}

{{< figure src="/images/posts/redhatsummit2018/sf2.jpeg" class="center" >}}

San Francisco skyline as viewed from the cruise.
