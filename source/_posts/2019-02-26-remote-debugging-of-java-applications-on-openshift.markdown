---
layout: post
title: "Remote Debugging of Java Applications on OpenShift"
date: 2019-02-26 13:08:30 -0800
comments: true
categories: development
---

In this article I am going to show you how to attach a debugger and a VisualVM profiler to the Java application running on OpenShift. The approach described here doesn't make use of the [Jolokia](https://jolokia.org/) bridge. Instead, we are going to leverage the port-forwarding feature of OpenShift.

<!-- more -->

The whole setup can be divided into three steps:

1. Enable debug and JMX ports on the JVM
2. Set up port forwarding
3. Attach debugger and VisualVM to the forwarded ports

I am going to use OpenShift v3.11 that I installed using Minishift and a test application built with Java OpenJDK 1.8. This is how the complete setup is going to look like:

{% img /images/posts/remote_debugging_of_java_applications_on_openshift.svg %}

## Hello world application

For those of you who want to follow along, let's set up a test application which we will use for debugging. If you already have your Java application running on OpenShift, you can jump ahead to the next section.

Let's deploy a Hello world application that I found on [GitHub](https://github.com/vert-x3/vertx-openshift-s2i). This application was originally created to demonstrate how to build [Vert.x](https://vertx.io/)-based microservices on OpenShift. You can get this application up and running in just two steps.

First, issue this command to build an S2I builder image for Vert.x applications:

{% codeblock lang:sh %}
$ oc create -f https://raw.githubusercontent.com/vert-x3/vertx-openshift-s2i/master/vertx-s2i-all.json
buildconfig.build.openshift.io/vertx-s2i created
imagestream.image.openshift.io/vertx-centos created
imagestream.image.openshift.io/vertx-s2i created
template.template.openshift.io/vertx-helloworld-maven created
{% endcodeblock %}

OpenShift started the build of the builder image and you can follow the progress with:

{% codeblock lang:sh %}
$ oc log -f bc/vertx-s2i

...

Removing intermediate container fc4bff8f426c
Successfully built bd4a858867e9
Pushing image 172.30.1.1:5000/myproject/vertx-s2i:latest ...
Pushed 1/8 layers, 50% complete
Pushed 2/8 layers, 25% complete
Pushed 3/8 layers, 38% complete
Pushed 4/8 layers, 50% complete
Pushed 5/8 layers, 63% complete
Pushed 6/8 layers, 97% complete
Pushed 7/8 layers, 99% complete
Pushed 8/8 layers, 100% complete
Push successful
{% endcodeblock %}

At the end of the build process OpenShift pushed the new image into the integrated Docker registry. Next, we are going to use the builder image to build and run a sample Vert.x application:

{% codeblock lang:sh %}
$ oc new-app vertx-helloworld-maven
--> Deploying template "myproject/vertx-helloworld-maven" to project myproject

     vertx-helloworld-maven
     ---------
     Sample Vert.x application build with Maven

     * With parameters:
        * APPLICATION_NAME=hello-world
        * APPLICATION_HOSTNAME=
        * GIT_URI=https://github.com/vert-x3/vertx-openshift-s2i.git
        * GIT_REF=master
        * CONTEXT_DIR=test/test-app-maven
        * APP_OPTIONS=
        * GITHUB_TRIGGER_SECRET=EM325a5K # generated
        * GENERIC_TRIGGER_SECRET=CBCcCIWr # generated

--> Creating resources ...
    buildconfig.build.openshift.io "hello-world" created
    imagestream.image.openshift.io "hello-world" created
    deploymentconfig.apps.openshift.io "hello-world" created
    route.route.openshift.io "hello-world" created
    service "hello-world" created
--> Success
    Build scheduled, use 'oc logs -f bc/hello-world' to track its progress.
    Access your application via route 'hello-world-myproject.192.168.42.115.nip.io'
    Run 'oc status' to view your app.
{% endcodeblock %}

You can follow the build logs by issuing the command:

{% codeblock lang:sh %}
$ oc log -f bc/hello-world

...

[INFO]
[INFO] --- maven-clean-plugin:2.5:clean (default-clean) @ vertx-hello-world ---
[INFO] Deleting /opt/app-root/src/source/target
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  1.102 s
[INFO] Finished at: 2019-02-26T20:21:57Z
[INFO] ------------------------------------------------------------------------
Application jar file is located in /opt/openshift/vertx-app.jar
Files located in the application directory:
total 13216
-rw-r--r--. 1 default root      286 Feb 26 20:21 additional_files.md
-rw-r--r--. 1 default root 13525420 Feb 26 20:21 vertx-app.jar
Pushing image 172.30.1.1:5000/myproject2/hello-world:latest ...
Pushed 0/9 layers, 2% complete
Pushed 1/9 layers, 11% complete
Push successful
{% endcodeblock %}

If everything went fine, you should be able to see the Hello world application running:

{% codeblock lang:sh %}
$ oc get pod | grep hello-world
hello-world-1-build   0/1       Completed    0          6m
hello-world-1-dw5lf   1/1       Running      0          42s
{% endcodeblock %}

## Enabling Debug and JMX ports on JVM

In the following, I am going to use OpenJDK 1.8. Note that the available JVM options may vary depending on the version of Java  platform you are using.

To enable a remote debug port on JVM, one has to pass the following option to the JVM:

{% codeblock %}
-agentlib:jdwp=transport=dt_socket,server=y,address=8000,suspend=n
{% endcodeblock %}

In order to enable JMX, the following JVM options are needed:

{% codeblock %}
-Dcom.sun.management.jmxremote=true
-Dcom.sun.management.jmxremote.port=3000
-Dcom.sun.management.jmxremote.rmi.port=3001
-Djava.rmi.server.hostname=127.0.0.1
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false
{% endcodeblock %}

This set of options deserves a bit more explanation. By default, JMX utilizes RMI as the underlying technology for the communication between the JMX client and the remote JVM. And as a matter of fact, there are two RMI ports needed for this communication:
* RMI registry port
* RMI server port

At the beginning, the client connects to the RMI registry on port 3000 and looks up the connection to the RMI server. After the successful lookup, the client initiates a second connection to the RMI server. Based on our configuration, the client is going to connect to 127.0.0.1:3001. However, there's no RMI server running on the local machine, so what's the deal? As you will see in the next section, we are going to forward the local port 3001 back to the remote server.

Next, we need to convey our configuration options to the JVM running inside the OpenShift pod. It turns out that there exists an environment variable `JAVA_TOOL_OPTIONS` that is interpreted directly by the JVM and where you can put your JVM configuration options. I recommend using this variable as there is a great chance that this variable will work no matter how deep in your wrapper scripts you are launching the JVM. Go ahead and modify the DeploymentConfig or Pod descriptor of your application in OpenShift to add the `JAVA_TOOL_OPTIONS` variable. For example, you can open the DeloymentConfig for editing like this:

{% codeblock lang:sh %}
$ oc edit dc hello-world
{% endcodeblock %}

... and add the `JAVA_TOOL_OPTIONS` environment variable to the container section of the specification:

{% codeblock %}
...

    spec:
      containers:
      - env:
        - name: JAVA_TOOL_OPTIONS
          value: -agentlib:jdwp=transport=dt_socket,server=y,address=8000,suspend=n
            -Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.port=3000
            -Dcom.sun.management.jmxremote.rmi.port=3001 -Djava.rmi.server.hostname=127.0.0.1
            -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false

...
{% endcodeblock %}

After applying the above changes, OpenShift will redeploy the application pod. At startup, JVM will print out the following line to the stderr which will show up in the container logs:


{% codeblock lang:sh %}
$ oc logs dc/hello-world | grep JAVA_TOOL_OPTIONS
Picked up JAVA_TOOL_OPTIONS: -agentlib:jdwp=transport=dt_socket,server=y,address=8000,suspend=n -Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.port=3000 -Dcom.sun.management.jmxremote.rmi.port=3001 -Djava.rmi.server.hostname=127.0.0.1 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false
{% endcodeblock %}

This verifies that our JVM options are in effect and the debug port and JMX ports are open. How are we going to connect to these ports? Let's set up port forwarding on the local machine next.

## Setting up port forwarding

OpenShift features [port forwarding](https://docs.okd.io/latest/dev_guide/port_forwarding.html) that allows you to connect to an arbitrary port of a pod running on OpenShift. Port forwarding doesn't require you to define any additional objects like Service or Route to enable it. What you need though is to start a port forwarding proxy on your local machine. Issue the following command on your local machine to start the proxy and forward the three ports 8000, 3000, and 3001 to the remote pod running on OpenShift:

{% codeblock lang:sh %}
$ oc port-forward <POD> 8000 3000 3001
{% endcodeblock %}

In the above command, remember to replace `<POD>` with the name of your application pod. If everything worked well, you should see the following output :

{% codeblock lang:sh %}
$ oc port-forward hello-world-2-55zlq 8000 3000 3001
Forwarding from 127.0.0.1:8000 -> 8000
Forwarding from 127.0.0.1:3000 -> 3000
Forwarding from 127.0.0.1:3001 -> 3001
{% endcodeblock %}

Note that the proxy keeps running on the foreground.

## Attaching to the JVM running on OpenShift

Having our port-forwarding proxy all set, let's fire up a debugger and attach it to our application. Note that we instruct the debugger to connect to the localhost on port 8000. This port is in turn forwarded to the port 8000 on the JVM:

{% codeblock lang:sh %}
$ jdb -connect com.sun.jdi.SocketAttach:hostname=localhost,port=8000
{% endcodeblock %}

After the debugger attaches, you can list existing JVM threads using the `threads` command:

{% codeblock %}
> threads
Group system:
  (java.lang.ref.Reference$ReferenceHandler)0x133a                                             Reference Handler                                   cond. waiting
  (java.lang.ref.Finalizer$FinalizerThread)0x133b                                              Finalizer                                           cond. waiting
  (java.lang.Thread)0x133c                                                                     Signal Dispatcher                                   running
  (java.lang.Thread)0x133d                                                                     RMI TCP Accept-3001                                 running
  (java.lang.Thread)0x133e                                                                     RMI TCP Accept-3000                                 running
  (java.lang.Thread)0x133f                                                                     RMI TCP Accept-0                                    running
Group main:
  (java.util.TimerThread)0x1342                                                                vertx-blocked-thread-checker                        cond. waiting
  (io.vertx.core.impl.VertxThread)0x1343                                                       vert.x-worker-thread-0                              cond. waiting

...
{% endcodeblock %}

Next, let's check out if we can attach [VisualVM](https://visualvm.github.io/) to our application as well:

{% codeblock lang:sh %}
$ visualvm --openjmx localhost:3000
{% endcodeblock %}

Works like a charm, doesn't it?

{% img /images/posts/visualvm_attached.png %}

## Conclusion

In this blog post, we were able to attach a debugger and VisualVM to the Java application running on OpenShift. We didn't need to deploy Jolokia proxy or create additional Service or Route objects to make our setup work. Instead, we leveraged the port-forwarding feature already available in OpenShift. The demonstrated method has additional security benefits as we are not exposing any additional ports of the application container.

Hope you enjoyed this article and was able to reproduce this setup for yourself. If you have any thoughts or questions feel free to add them to the comment section below.
