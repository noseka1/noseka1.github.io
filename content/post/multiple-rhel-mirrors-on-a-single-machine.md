+++
title = "Multiple RHEL Mirrors on a Single Machine"
date = "2015-08-16"
slug = "2015/08/16/multiple-rhel-mirrors-on-a-single-machine"
Categories = [ "devops" ]
+++

*Reposync* can mirror the yum repository to which your machine is subscribed to. However, you cannot subscribe your machine to the RHEL6 and RHEL7 at the same time. Let's take a look at how Docker can help us here.

<!--more-->

The RHEL Docker images are available in the public registry. For example, the following command downloads a RHEL6 Docker image:

{{< highlight shell "linenos=table" >}}
$ docker pull rhel6
{{< / highlight >}}

The downloaded image is not registered with [Red Hat Customer Portal](https://access.redhat.com/ "Red Hat Customer Portal") and hence cannot receive any software updates. To register the Docker image, you'll need the *username* and *password* you use to log in to the Red Hat Customer Portal.

# Docker and the build-time secrets

You will encounter a problem when building and registering the RHEL image using a Dockerfile. How to register the RHEL image using your credentials without Docker baking those credentials into the image? You don't want others to discover your secrets in the image's metadata or the history log. As a matter of fact, there doesn't seem to be a secure way to pass the secrets to the Docker in build-time. And there is a nice [write-up](https://github.com/docker/docker/issues/13490 "Secrets in Docker") available in the Docker issue tracker describing the different approaches how to workaround this deficiency.

The following approach worked for me:

1. Store the credentials into a file on the Docker host
2. Start a RHEL container with the credentials file mounted into it
3. Use the credentials to register the RHEL system
4. Stop the RHEL container
5. Commit the RHEL container into a new Docker image

# Creating RHEL images with Ansible

You can find the `rhel_reposync` Ansible role at [GitHub](https://github.com/noseka1/rhel_reposync "rhel_reposync"). You want to run this role on your Docker host. It will download the RHEL6 and RHEL7 images, and register them with the Red Hat Customer Portal. You have to supply your Red Hat credentials on the command-line. For example:

{{< highlight shell "linenos=table" >}}
$ ansible-playbook -i hosts playbook.yml -e redhat_portal_username=username@company.com -e redhat_portal_password=secretpassword
{{< / highlight >}}

After the Ansible run is finished you'll find two images in your local Docker repository: `rhel6_reposync_registered` and `rhel7_reposync_registered`.

# Mirroring the RHEL repository

To begin the RHEL repository mirroring you just start a container based on the registered image. The container expects a volume to be mounted at `/repodir`. It will save the downloaded RPM packages at this location.

{{< highlight shell "linenos=table" >}}
$ docker run --rm -v /var/www/html/RHEL6:/repodir rhel6_reposync_registered
{{< / highlight >}}

# Alternative solution

I discovered an [upstream_sync](https://github.com/pyther/upstream_sync "upstream_sync") script that allows you to mirror multiple RHEL/CentOS repositories from a single machine. It uses a specific client SSL certificate and key to access each of the repositories. This solution is much simpler than the approach described in this article.
