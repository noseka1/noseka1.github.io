+++
title = "An Introduction to Building on Openshift"
date = "2017-03-19"
slug = "2017/03/19/an-introduction-to-building-on-openshift"
Categories = []
+++

For years a Jenkins server has been driving the software builds in our company. Some time ago, we deployed an OpenShift cluster. The primary purpose of our OpenShift cluster was to support the efforts of dockerizing our software products. However, as OpeShift is a complete PaaS solution we started thinking about leveraging OpenShift for software builds, too. In this blog post I'd like to share what we learned about building on OpenShift so far.

<!-- more -->

Before we begin talking about OpenShift, let's briefly discuss our current build environment. In the center of our build environment there is a Jenkins server. In Jenkins, we maintain numerous jobs to build our software, run automated tests, drive various devops tasks and much more. Jenkins is our central place from where the automated processes are started and monitored. As Jenkins is greatly extensible via plugins, we were able to easily integrate Jenkins with other tools, too.

## Building the OpenShift way

{% img right /images/posts/openshift_logo.gif 200 200 %}

In OpenShift, one has to create a *BuildConfig* resource to describe the [build process](https://docs.openshift.org/latest/dev_guide/builds/index.html). The BuildConfig resource in OpenShift is roughly equivalent to a job definition in Jenkins. When creating a BuildConfig resource, a build strategy has to be chosen. The build strategy resembles a job type in Jenkins. Currently, there are four build strategies available in OpenShift:

**Source-to-Image strategy**. Allows you to create a container image starting from the application source code. During the build process, the source code is downloaded into a container and compiled there. The finished binary artifacts are installed into the container. The complete container image is then pushed into the Docker registry from where it can be deployed as an application on OpenShift.

**Docker strategy**. The input of the build process is a Dockerfile. OpenShift will execute a Docker build using the provided Dockerfile and upload the resulting image into the Docker registry from where it can be deployed.

**Custom strategy**. Custom strategy could be compared to a free style job in Jenkins. The outcome of the build doesn't have to be a Docker image. Instead, the custom strategy allows you to create JARs, tarballs, RPMs or other artifacts which you have to upload to the repository of your choice by the end of the build.

**Pipeline strategy**. In OpenShift 3.3, a new build strategy was introduced called *Pipeline*. This strategy doesn't really build anything but enables you to implement workflows on OpenShift. The great article [OpenShift 3.3 Pipelines - Deep Dive](https://blog.openshift.com/openshift-3-3-pipelines-deep-dive/) describes how the Pipeline strategy works. In summary, you can create a BuildConfig in OpenShift that contains a definition of a Jenkins pipeline (using the Groovy DSL language). Based on this definition, OpenShift will create a pipeline job in Jenkins and execute it. Among other things, the Jenkins job can trigger a build on OpenShift, verify that the build succeeded and trigger a deployment. This approach allows OpenShift to leverage Jenkins pipelines to orchestrate a more involved CI/CD workflow possibly encompassing a conditional execution of multiple OpenShift builds and deployments.

An alternative to using the Pipeline strategy in OpenShift would be defining the pipeline job directly in Jenkins. With the [OpenShift pipeline plugin](https://plugins.jenkins.io/openshift-pipeline) installed, one can trigger OpenShift operations from within the pipeline job.

As I didn't really work with the OpenShift strategies much I'm not going to elaborate any further. Instead, in the next section, I'm going to mention two Jenkins plugins that we are successfully using to run builds on OpenShift.

## Builds on Openshift driven by Jenkins

{% img right /images/posts/jenkins_logo.png 200 200 %}

There are two Jenkins plugins that can leverage OpenShift containers as build slaves:

**[Swarm plugin](https://wiki.jenkins-ci.org/display/JENKINS/Swarm+Plugin)**. The Swarm plugin consists of two parts: a Jenkins plugin and a CLI client. Jenkins plugin exposes an endpoint where the CLI clients can register themselves. A CLI client acts as a Jenkins slave. It runs indefinitely within a Docker container and provides Jenkins with a configurable number of build executors. While the plugin is called a Swarm plugin it doesn't really need any Swarm orchestration. It can happily run in a Docker container on OpenShift.

**[Kubernetes plugin](https://wiki.jenkins-ci.org/display/JENKINS/Kubernetes+Plugin)**. Works perfectly with OpenShift. In contrast to the Swarm plugin, Kubernetes plugin spins up a new Docker slave for each job on the fly and destroys it as soon as the job has finished running.

Because the Jenkins workspace is created inside of the container, it will be deleted as soon as the Docker container is terminated. If you'd like to reuse the same workspace for subsequent builds, I'd like to offer you two options how to create persistent workspaces:

1. You can attach a volume of type *hostPath* to your slave pods and place your workspace on that volume. At the same time you have to speficy a *nodeSelector* on your slave pods that would instruct OpenShift to schedule all your slave pods onto the same OpenShift node. With this approach the Jenkins slave can access its workspace on the local storage. Unfortunately, all the slaves that need to share a workspace have to run on the same OpenShift node which can get overloaded.

2. You can attach a volume with the *ReadWriteMany* capability to your slave pods and place your workspace on this volume. Eligible volume types are NFS, GlusterFS or CephFS. Using this method a Jenkins slave running on any node in the cluster can access the shared workspace. The downside is that the access is over the network and hence slower than an access to the local storage.

## Conclusion

In the this blog post we reviewed different approaches how to leverage an OpenShift cluster for software builds. On one hand, builds can be defined within OpenShift by creating the BuildConfig resources. This approach might be less flexible than using a full-fledged build server like Jenkins, however, one can be sure that the builds will work on any OpenShift cluster including the public cloud. On the other hand, we have seen that in an environment where Jenkins is already the king, we can leverage the Swarm or Kubernetes plugin to allow Jenkins to schedule build jobs on OpenShift.
