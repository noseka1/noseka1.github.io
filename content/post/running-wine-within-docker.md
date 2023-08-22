+++
title = "Running Wine Within Docker"
date = "2015-07-04"
slug = "2015/07/04/running-wine-within-docker"
Categories = [ "devops" ]
+++

After upgrading to Debian Jessie, my Windows application running under Wine stopped working. In this article we'll use Docker to restore the Wine environment from Debian Wheezy. We'll run the Windows application inside this Docker container.

<!--more-->

Docker is not part of the stable Jessie distribution, however, you can install it from the [Debian Backports](http://backports.debian.org/ "Debian Backports") repositories.

## Creating a Docker image

We start off with creating a Docker image based on the `debian:wheezy` image from the official Docker repositories. We'll install the 32-bit Wine package on it. The Wine application is a graphical application and hence requires access to the X server. Setting the environmet variable `DISPLAY=:0` instructs the application to access the local X server. The complete `Dockerfile` to build our Wine image looks as follows:

{{< highlight-caption lang="docker" linenos="table" title="Dockerfile" >}}
FROM debian:wheezy
RUN dpkg --add-architecture i386
RUN apt-get update
RUN apt-get install --no-install-recommends --assume-yes wine
ENV DISPLAY :0
{{< / highlight-caption >}}

You can kick off the build of the `wine1.4` image with:

{{< highlight shell "linenos=table" >}}
$ docker build --rm -t wine1.4 .
{{< / highlight >}}

After a minute or two the build is complete and the resulting image is stored locally on your Docker host. You can take a look using the `docker images` command:
{{< highlight shell "linenos=table" >}}
$ docker images | grep wine1.4
wine1.4                   latest              b300b8573303        About a minute ago   271.3 MB
{{< / highlight >}}

## Running Wine within a Docker container

To test our `wine1.4` Docker image, we'll run the `notepad` application which comes with Wine:

{{< highlight shell "linenos=table" >}}
$ docker run --rm wine1.4 wine "C:\windows\system32\notepad.exe"
wine: created the configuration directory '/root/.wine'
Application tried to create a window, but no driver could be loaded.
Make sure that your X server is running and that $DISPLAY is set correctly.
err:systray:initialize_systray Could not create tray window
Application tried to create a window, but no driver could be loaded.
Make sure that your X server is running and that $DISPLAY is set correctly.
Application tried to create a window, but no driver could be loaded.
Make sure that your X server is running and that $DISPLAY is set correctly.
Application tried to create a window, but no driver could be loaded.
Make sure that your X server is running and that $DISPLAY is set correctly.

....
{{< / highlight >}}

The `notepad` application doesn't really start. From the generated error output we can see that our application is unable to access the X server. Well, there's no X server running inside the container. In order to allow the application running inside the container to access the X server running on the Docker host, we'll expose the host's X server UNIX domain socket inside the container. We can ask Docker to bind mount the `/tmp/.X11-unix/X0` UNIX socket to the same location inside the container using the `--volume` parameter:

{{< highlight shell "linenos=table" >}}
$ docker run --rm --volume /tmp/.X11-unix/X0:/tmp/.X11-unix/X0 wine1.4 wine "C:\windows\system32\notepad.exe"
{{< / highlight >}}

When we run the above command, Wine starts up successfully and the notepad application opens up. Inside the Docker container Wine runs as user root and starts from scratch with no existing configuration:

{% img center /images/posts/wine.png %}

We would like Wine running inside the Docker container to use the existing Wine configuration stored on the Docker host. Let's copy the existing Wine configuration on the host to a new directory which we'll in turn expose inside the Docker container:

{{< highlight shell "linenos=table" >}}
$ cd ~
$ cp -a .wine .wine.docker
{{< / highlight >}}

The `.wine.docker` directory can be exposed inside the Docker container  with the command-line parameter `--volume /home/anosek/.wine.docker:/home/anosek/.wine`. The Wine configuration in `.wine.docker` is in my case owned by the user `anosek`. We want Docker to run as user `anosek` instead of the default `root` user. In order to accomplish this, two additional parameters to the Docker `run` command are needed: `--volume /etc/passwd:/etc/passwd` and `--user anosek`. We're bind mounting the `/etc/passwd` file including the definition of user `anosek` inside the Docker container and asking Docker to run as user `anosek`.

The complete command to run Wine inside the Docker container as user `anosek` and using the existing Wine configuration found in the `/home/anosek/.wine.docker` directory on the host looks as follows:

{{< highlight shell "linenos=table" >}}
$ docker run --rm --volume /tmp/.X11-unix/X0:/tmp/.X11-unix/X0 --volume /home/anosek/.wine.docker:/home/anosek/.wine --volume /etc/passwd:/etc/passwd --user anosek wine1.4 wine "C:\windows\system32\notepad.exe"
{{< / highlight >}}

## Conclusion

One can leverage the modern Docker technology to create a specific Wine environment on any Linux system. To achieve this a little bit of configuration is required, though. In case of Wine you might be better off using some of the specialized tools for Wine management. For instance, [PlayOnLinux](https://www.playonlinux.com/ "PlayOnLinux") can manage different versions of Wine as well as different prefixes (`WINEPREFIX` environment variable).
