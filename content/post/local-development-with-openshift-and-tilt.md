+++
title = "Local Development with Openshift and Tilt"
date = "2020-06-08"
slug = "2020/06/08/local-development-with-openshift-and-tilt"
Categories = [ "development" ]
+++

In this blog post, I am going to show you how to use Tilt to facilitate local OpenShift development. Tilt’s capabilities will be demonstrated in a practical example that uses buildah and CodeReady Containers. If you develop containerized applications on OpenShift, this blog post is for you.

<!--more-->

## How does Tilt facilitate local development?

The diagram below depicts a development workflow orchestrated by Tilt. After you execute the `tilt up` command on your development machine, Tilt will keep running while performing the following actions:

1. Tilt watches for changes in the source code made by the developer on the local machine.
2. After a change has been detected, Tilt executes buildah to update the container image. After the build completes, the updated container image is pushed to the internal registry in CodeReady Containers.
3. Tilt watches Kubernetes manifests on the local machine and keeps them in sync with CodeReady Containers. Any changes made to the manifests are instantly applied to CodeReady Containers.
4. Tilt forwards local ports to the application pod running in CodeReady Containers. This allows the developer to conveniently access the application on localhost.

{{< figure src="/images/posts/local_development_with_openshift_and_tilt_diagram.png" class="center" >}}

Tilt helps the developer automate many of the manual steps made during the development of containerized applications. It speeds up the edit-compile-run loop considerably. Interested to trying it out? Follow me to the next section, where we will implement the workflow depicted in the above diagram.

## Using Tilt for developing a sample application

In this section, we are going to use Tilt to orchestrate the development of a sample application. As I didn't want to reinvent the wheel by designing a custom application, I grabbed the Plain Old Static HTML example that comes with Tilt and can be found on [GitHub](https://github.com/tilt-dev/tilt-example-html/tree/faad605963b396b0863151802544fb01f6b414c6/0-base). This example is described in the Tilt's [documentation](https://docs.tilt.dev/example_static_html.html) and you may already be familiar with it. It consists of a very simple shell script that serves a static HTML. In contrast to Tilt's example which leverages Docker and upstream Kubernetes, I will be using developer tools from the Red Hat's portfolio:

* [buildah](https://buildah.io/)
* [UBI container images](https://www.redhat.com/en/blog/introducing-red-hat-universal-base-image)
* [CodeReady Containers](https://developers.redhat.com/products/codeready-containers)

In order to be able to use the Red Hat developer tools, I had to modify the original sample code. The sample code used in this tutorial can be found on [GitHub](https://github.com/noseka1/local-development-with-openshift-and-tilt). I recommend that you go ahead and briefly review it.

The overall setup consists of a Fedora 32 development machine where I installed buildah and CoreReady Containers (CRC) version 1.10. I installed Tilt version 0.13.4 which is the latest release of Tilt available at the time of this writing. If you plan to use Tilt along with CodeReady Containers, I recommend grabbing this or any future versions of Tilt, as this version includes a [patch]( https://github.com/windmilleng/tilt/commit/7e9487816ea32ed086318ce7373c67d9febb6f36) that makes Tilt work with CodeReady Containers without the need for further configuration. The Tilt binary can be downloaded from [GitHub](https://github.com/tilt-dev/tilt/releases).

Having the required tools in place, let's start by logging in into CRC and creating a new project:

{{< highlight shell "linenos=table" >}}
$ oc login
$ oc new-project tilt-example
{{< / highlight >}}

Let's now focus  on configuring buildah to be able the pull and push images from the container registries. As you can see in the [Dockerfile](https://github.com/noseka1/local-development-with-openshift-and-tilt/blob/master/Dockerfile), our application uses the `registry.redhat.io/ubi8/ubi` container image as the base image. In order for buildah to be able to pull this image from the registry, we need to log in to this registry using the Red Hat Customer Portal credentials:

{{< highlight shell "linenos=table" >}}
$ buildah login registry.redhat.io
{{< / highlight >}}

Next, let's configure buildah to be able to push images into the CRC internal registry. The CRC registry endpoint uses a self-signed certificate. Buildah will refuse to communicate with the internal registry as the certificate is signed by an unknown authority. In order for buildah to be able to push images into the internal registry, you will need to add this registry to the list of insecure registries. On your development machine, edit `/etc/containers/registries.conf` and add the CRC internal registry to the list of insecure registries.

{{< highlight toml "linenos=table" >}}
[registries.insecure]
registries = [ 'default-route-openshift-image-registry.apps-crc.testing' ]
{{< / highlight >}}

In order for buildah to be able to push images into the CRC registry, we need to log in to this registry. For that, use the `oc` command to grab a token used for authentication against the registry:

{{< highlight shell "linenos=table" >}}
$ oc whoami --show-token
7geTDzA6Mqa-NeXweTXtOFUJtEHocVShKl5yxtxqeB0
{{< / highlight >}}

Log in to the registry using the authentication token:

{{< highlight shell "linenos=table" >}}
$ buildah login \
    --username unused \
    --password 7geTDzA6Mqa-NeXweTXtOFUJtEHocVShKl5yxtxqeB0 \
    default-route-openshift-image-registry.apps-crc.testing
Login Succeeded!
{{< / highlight >}}

After successfully logging into the CRC internal registry, the buildah configuration is now complete. Finally, we can turn our attention to Tilt. First, let's review the `Tiltfile` which describes how Tilt will orchestrate our development workflow:

{{< highlight plaintext "linenos=table" >}}
# push the container image to the CRC internal registry, project tilt-example
default_registry(
  'default-route-openshift-image-registry.apps-crc.testing/tilt-example',
  host_from_cluster='image-registry.openshift-image-registry.svc:5000/tilt-example')

# use buildah to build and push the container image
custom_build(
  'example-html-image',
  'buildah build-using-dockerfile --tag $EXPECTED_REF . && buildah push $EXPECTED_REF',
  ['.'],
  skips_local_docker=True)

# deploy Kubernetes resource
k8s_yaml('kubernetes.yaml')

# make the application available on localhost:8000
k8s_resource('example-html', port_forwards=8000)
{{< / highlight >}}

I annotated the `Tiltfile` with comments that explain the meaning of individual Tilt instructions. Ready to give it a shot? Just clone the git repository and run Tilt:

{{< highlight shell "linenos=table" >}}
$ git clone https://github.com/noseka1/local-development-with-openshift-and-tilt
$ cd local-development-with-openshift-and-tilt/
$ tilt up
{{< / highlight >}}

After Tilt comes up, it will call buildah to pull the base image, build the application, and push the resulting image to the CRC internal registry. It will also deploy the application on Kubernetes by applying the `kubernetes.yaml` manifest referenced in the `Tiltfile`. If everything worked well, and the application pod starts up, you will see the "Serving files on port 8000" log message in the bottom pane:

{{< figure src="/images/posts/local_development_with_openshift_and_tilt.png" class="center" >}}

At this point, you should be able to reach the running application on `localhost:8000`:

{{< highlight shell "linenos=table" >}}
$ curl localhost:8000
<!doctype html>
<html>
  <body style="font-size: 30px; font-family: sans-serif; margin: 0;">
    <div style="display: flex; flex-direction: column; width: 100vw; height: 100vh; align-items: center; justify-content: center;">
      <img src="pets.png" style="max-width: 30vw; max-height: 30vh;">
      <div>Hello cats!</div>
    </div>
  </body>
</html>
{{< / highlight >}}

To experience how Tilt facilitates the local development, change the content of the `index.html` file. After you save your changes, Tilt will instantly re-run the loop and deploy the updated application.

## Conclusion

In this blog, we described how Tilt can facilitate the local development of containerized applications. We demonstrated Tilt's capabilities in a practical example that showed Tilt working along with buildah and CodeReady Containers. In this introductory article, we were able to only scratch the surface. There is much more that Tilt has to offer, including live updates that can update the application without restarting the container, and which can drastically speed up the edit-compile-run loop. I encourage you to read through [Tilt's documentation](https://docs.tilt.dev/) to learn more about this tool.

Do you use Tilt for development on OpenShift or Kubernetes? What's your opinion on Tilt? I would be happy to hear about your experiences. You can leave your comments in the comment section below.
