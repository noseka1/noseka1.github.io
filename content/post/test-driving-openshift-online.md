+++
title = "Test Driving Openshift Online"
date = "2017-08-27"
slug = "2017/08/27/test-driving-openshift-online"
Categories = [ "cloud" ]
+++

Did you decide to dockerize your application? Awesome! Are you looking for a place to build and host your Docker containers? OpenShift Online is a service that allows you to build and run your Docker containers. Read on, if you want to learn more about it.

<!--more-->

[OpenShift Online](https://manage.openshift.com/) is a Platform-as-a-Service cloud managed by Red Hat. It is hosted on Amazon Web Services (AWS). OpenShift Online is a multi-tenant offering. That means that containers owned by different tenants are running on the same cluster. If you're more interested in renting your own private OpenShift cluster, you shall consider the [OpenShift Dedicated](https://www.openshift.com/dedicated/index.html) offering, instead.

The information about the plans and pricing of the OpenShift Online service can be found [here](https://www.openshift.com/pricing/index.html). In this blog post, we're going to utilize the free of charge *Starter* plan. With this plan, you get 1GiB of memory for your containers, another 1GiB of memory for running builds, deployments and jobs, and finally 1GiB of persistent storage. The real limitation of the free *Starter* plan is the resource hibernation. Your containers will be forced to sleep 18 hours in a 72 hour period.

## Welcome to OpenShift Online

You can login into OpenShift Online [here](https://manage.openshift.com/). If you don't have an OpenShift account, the sign up and creation of the OpenShift account doesn't take you long. After you login into your OpenShift account, find a small question mark icon in the top right corner of the welcome page. From the drop down menu choose the "Command Line Tools" option.

{% img left /images/posts/openshift_online/openshift_online_welcome.png %}

Download the `oc` client tool using one of the provided download links. We're going to use the `oc` tool throughout this tutorial. After downloading the distribution archive, extract the `oc` tool from it and place it somewhere on your PATH. The `oc` tool is a single statically linked binary which makes the installation straight forward. On the same page, notice the instructions on how to login into the CLI. We'll be logging into it in a moment.

{% img left /images/posts/openshift_online/openshift_online_command_line_tools.png %}

Go back to the welcome page and this time choose the "About" option from the drop down menu in the top right corner. On the "About" page, you can learn that OpenShift Online is built upon a fairly recent version of OpenShift. That's great to know, as each new version of OpenShift comes with a ton of new features. As the OpenShift Online user you would not like to be left behind with an older version of OpenShift.

On the same "About" page, you can find the address of the integrated Docker registry. You can push ready images to this registry in order to make them available for deployment on OpenShift. If you build images on OpenShift, the finished images will be pushed into this registry at the end of the build process. In addition to deploying images from the integrated Docker registry, you can, of course, deploy images directly from the Docker Hub as well.

{% img left /images/posts/openshift_online/openshift_online_about.png %}

## Getting our hands dirty

In this section, we're going to use the CLI tool to exercise the OpenShift Online functionality. From the "Command Line Tools" page that we visited previously, copy the command to login into the CLI tool:

{{< highlight shell "linenos=table" >}}
$ oc login https://api.starter-us-west-1.openshift.com --token=6BoQ7DQ8nUeEE3FOWzvDCgcUD0T6LdBBZEuUiPlD7Tc
Logged into "https://api.starter-us-west-1.openshift.com:443" as "anosek@example.com" using the token provided.

You don't have any projects. You can try to create a new project, by running

    oc new-project <projectname>
{{< / highlight >}}

Note that the value of your login token will differ. Next, let's create a new OpenShift project called `php-hello`:

{{< highlight shell "linenos=table" >}}
$ oc new-project php-hello
Now using project "php-hello" on server "https://api.starter-us-west-1.openshift.com:443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app centos/ruby-22-centos7~https://github.com/openshift/ruby-ex.git

to build a new example application in Ruby.
{{< / highlight >}}

The Starter plan allows you to create only a single project. Next, we're going to build our PHP application. The source code of the application can be found at [GitHub](https://github.com/noseka1/openshift-php-hello). The whole purpose of the application is to return a Hello message containing the name of the host that the application is running on. You can build the application using the `oc new-build` command:

{{< highlight shell "linenos=table" >}}
$ oc new-build https://github.com/noseka1/openshift-php-hello.git
--> Found image 0d2f8fa (4 weeks old) in image stream "openshift/php" under tag "7.0" for "php"

    Apache 2.4 with PHP 7.0
    -----------------------
    PHP 7.0 available as docker container is a base platform for building and running various PHP 7.0 applications and frameworks. PHP is an HTML-embedded scripting language. PHP attempts to make it easy for developers to write dynamically generated web pages. PHP also offers built-in database integration for several commercial and non-commercial database management systems, so writing a database-enabled webpage with PHP is fairly simple. The most common use of PHP coding is probably as a replacement for CGI scripts.

    Tags: builder, php, php70, rh-php70

    * The source repository appears to match: php
    * A source build using source code from https://github.com/noseka1/openshift-php-hello.git will be created
      * The resulting image will be pushed to image stream "openshift-php-hello:latest"
      * Use 'start-build' to trigger a new build

--> Creating resources with label build=openshift-php-hello ...
    imagestream "openshift-php-hello" created
    buildconfig "openshift-php-hello" created
--> Success
    Build configuration "openshift-php-hello" created and build triggered.
    Run 'oc logs -f bc/openshift-php-hello' to stream the build progress.
{{< / highlight >}}

OpenShift automatically detects, that we're building a PHP application and will use an appropriate build image. The build image contains a pre-installed Apache server with mod_php. During the build, the `index.php` file from the source code repository is copied into the document root of the Apache server. The build process takes a minute or two to complete. You can follow the progress of your build using the `oc logs` command:

{{< highlight shell "linenos=table" >}}
$ oc logs -f bc/openshift-php-hello
Cloning "https://github.com/noseka1/openshift-php-hello.git" ...
        Commit: 35d7f33ca8180b9a331d9bd5ce4735c941da9d03 (Add index.php)
        Author: Ales Nosek <ales.nosek@gmail.com>
        Date:   Mon Aug 28 13:50:56 2017 -0700
Pulling image "registry.access.redhat.com/rhscl/php-70-rhel7@sha256:1f12d421bfc18874c5a7fdc41634ca5dd1cbd955c437738d571088e65dd0ba51" ...
---> Installing application source...

Pushing image 172.30.148.65:5000/php-hello/openshift-php-hello:latest ...
Pushed 0/6 layers, 7% complete
Pushed 1/6 layers, 18% complete
Pushed 2/6 layers, 37% complete
Pushed 3/6 layers, 56% complete
Pushed 4/6 layers, 79% complete
Pushed 5/6 layers, 99% complete
Pushed 6/6 layers, 100% complete
Push successful
{{< / highlight >}}

The output of the build is a new `openshift-php-hello` Docker image. This image is automatically pushed into the integrated Docker registry by OpenShift. In the next step, we're going to create a DeploymentConfig file which describes how to deploy our brand new `openshift-php-hello` image on OpenShift. Create a file named `php-hello.yml` with the following content:

{{< highlight-caption lang="yaml" linenos="table" title="php-hello.yml" >}}
kind: "DeploymentConfig"
apiVersion: "v1"
metadata:
  name: "php-hello"
spec:
  template:
    metadata:
      labels:
        name: "php-hello"
    spec:
      containers:
        - name: "php-hello"
          image: ""
          imagePullPolicy: "Always"
          ports:
            - containerPort: 8080
  replicas: 1
  selector:
    name: "php-hello"
  triggers:
    - type: "ConfigChange"
    - type: "ImageChange"
      imageChangeParams:
        automatic: true
        containerNames:
          - "php-hello"
        from:
          kind: "ImageStreamTag"
          name: "openshift-php-hello:latest"
  strategy:
    type: "Rolling"
{{< / highlight-caption >}}

Submit this file to OpenShift in order to launch the deployment:

{{< highlight shell "linenos=table" >}}
$ oc create -f php-hello.yml
deploymentconfig "php-hello" created
{{< / highlight >}}

Based on the DeploymentConfig descriptor, OpenShift will start one container `php-hello-1-XXXXX`. You can check whether the container is running with:

{{< highlight shell "linenos=table" >}}
$ oc get pod
NAME                          READY     STATUS      RESTARTS   AGE
openshift-php-hello-1-build   0/1       Completed   0          2h
php-hello-1-zw48t             1/1       Running     0          2m
{{< / highlight >}}

Verify that the `READY` column for your `php-hello` container eventually reads `1/1`. This indicates, that the deployment was successful and your container is running. In the following steps, we're going to configure OpenShift routing to allow the external network traffic to reach our container. For further information about the traffic routing on OpenShift, you can refer to my older blog post [Accessing Kubernetes Pods from Outside of the Cluster](/blog/2017/02/14/accessing-kubernetes-pods-from-outside-of-the-cluster/). First, let's create a service for our `php-hello` container:

{{< highlight shell "linenos=table" >}}
$ oc expose dc php-hello --port 8080
service "php-hello" exposed
{{< / highlight >}}

The created service functions as a internal load balancer. It forwards the traffic to the `php-hello` container on port 8080. The internal IP address of this load balancer is `172.30.80.50`, as we can learn when listing the existing services:

{{< highlight shell "linenos=table" >}}
$ oc get svc
NAME        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
php-hello   172.30.80.50   <none>        8080/TCP   4s
{{< / highlight >}}

Finally, we're going to create a route which will forward the external traffic to our service and hence to our `php-hello` container:

{{< highlight shell "linenos=table" >}}
$ oc expose svc php-hello
route "php-hello" exposed
{{< / highlight >}}

The route is assigned a public FQDN `php-hello-php-hello.a3c1.starter-us-west-1.openshiftapps.com`. As we haven't configured the route to use the TLS protocol, our application will be reachable on the standard HTTP port 80. The assigned FQDN can be found in the output of the `oc get route` command:

{{< highlight shell "linenos=table" >}}
$ oc get route
NAME        HOST/PORT                                                      PATH      SERVICES    PORT      TERMINATION   WILDCARD
php-hello   php-hello-php-hello.a3c1.starter-us-west-1.openshiftapps.com             php-hello   8080                    None
{{< / highlight >}}

At this moment, our application should be reachable over the public Internet. Let's send an HTTP request to our application:

{{< highlight shell "linenos=table" >}}
$ curl php-hello-php-hello.a3c1.starter-us-west-1.openshiftapps.com
Hello from php-hello-1-zw48t!
{{< / highlight >}}

Excellent, the application responded with the Hello message as expected. In the last exercise of our tutorial, we're going to scale our application. Let's ask OpenShift to create one more `php-hello` container:

{{< highlight shell "linenos=table" >}}
$ oc scale dc php-hello --replicas 2
deploymentconfig "php-hello" scaled
{{< / highlight >}}

The deployment of the second container will complete shortly. You can check the status of running containers using the `oc get pod` commmand:

{{< highlight shell "linenos=table" >}}
$ oc get pod
NAME                          READY     STATUS      RESTARTS   AGE
openshift-php-hello-1-build   0/1       Completed   0          2h
php-hello-1-cl5nw             1/1       Running     0          30s
php-hello-1-zw48t             1/1       Running     0          21m
{{< / highlight >}}

At this point, we're having two Docker containers ready to serve our HTTP requests. Let's generate some traffic and observe how OpenShift load balances the incoming requests between the two containers:

{{< highlight shell "linenos=table" >}}
$ curl php-hello-php-hello.a3c1.starter-us-west-1.openshiftapps.com
Hello from php-hello-1-zw48t!
$ curl php-hello-php-hello.a3c1.starter-us-west-1.openshiftapps.com
Hello from php-hello-1-cl5nw!
$ curl php-hello-php-hello.a3c1.starter-us-west-1.openshiftapps.com
Hello from php-hello-1-zw48t!
$ curl php-hello-php-hello.a3c1.starter-us-west-1.openshiftapps.com
Hello from php-hello-1-cl5nw!
{{< / highlight >}}

## Conclusion

OpenShift is a feature-rich container platform. In this blog post, we were only able to scratch the surface.

Are you considering to host your containerized application on OpenShift? Or are you already running your production apps on OpenShift? I would like to hear your experiences, please, leave your comments below.
