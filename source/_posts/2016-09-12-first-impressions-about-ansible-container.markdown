---
layout: post
title: "First Impressions about ansible-container"
date: 2016-09-12 20:06:49 -0700
comments: true
categories: devops
---

At Red Hat summit I learned about the new project [ansible-container](https://www.ansible.com/ansible-container). I was very excited and looked forward to building Docker containers with Ansible instead of the Dockerfiles. The project seemed to came just on time as in our company we're starting to Dockerize our software products. How did *ansible-container* work out for us? Read on!

<!-- more -->

The ansible-container is a very young project. The first release 0.1.0 came out on July 28, 2016. The project's Git repository is hosted on [GitHub](https://github.com/ansible/ansible-container). Additionaly, there's a repository with examples of how to use ansible-container available [here](https://github.com/ansible/ansible-container-examples).

## How ansible-container works

Here is how you build Docker containers with ansible-container. After crafting your Ansible playbook, you run the command `ansible-container build` which roughly executes these steps:

1. Spin up a *builder container* and mount your Ansible playbook directory as a volume inside of this container.
2. Spin up a *base container* from the Docker image of your choice.
3. Run ansible-playbook inside of the *builder container*. Ansible will connect to the *base container* and configure it accordingly to the playbook.
4. Shutdown the *base container* and commit it as a new image.

Besides building one Docker image at a time, it's rather straight forward to build a group of images at once. Under the hood, ansible-container leverages Docker Compose to start any number of containers to configure them with Ansible.

After your images are built, you can start them as containers on the local machine using the `ansible-container run` command. And finally, the `ansible-container shipit` command can help you to deploy your containers to Kubernetes or OpenShift.

## Evaluating ansible-container

Software development in our company is organized in a way that can probably be found in many other places, too. We maintain base software libraries in a project called *platform*. On top of *platform*, many different product components are developed. We distribute our software in the form of RPM packages. As a first step on our way towards containers, we are going to install the RPMs on top of the Docker base images.

After spending a week with ansible-container, we decided that we are not going to adopt it at this time. The main reasons were:

1. We would have to introduce another dependency to our build process. Our build machines would need to have Ansible, ansible-container and Docker Compose installed in order to build Docker containers. We prefer to keep our build dependencies to the minimum.

2. We need to build around 20 containers. During the container build we run a shell script to install an RPM package and to do a few more modifications. Our container build process is rather simple and Dockerfiles can just get the job done. For a more involved build process, Ansible would be preferable.

3. The ansible-container project shows its potential while it's still in its infancy. The builds with ansible-container take longer than the Dockerfile builds that leverage caching of image layers. The build specification file `container.yml` is not flexible enough for us. For example, it's not easy to build only a subset of the declared containers. Also it's not possible to build in one pass containers which inherit from each other.

## Conclusion

The ansible-container project is definitely very promising and we will keep our eye on it. For now, we're moving forward with generating and running Dockerfiles.
