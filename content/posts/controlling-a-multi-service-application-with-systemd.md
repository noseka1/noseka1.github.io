+++
title = "Controlling a Multi Service Application With Systemd"
date = "2016-12-04"
slug = "2016/12/04/controlling-a-multi-service-application-with-systemd"
Categories = []
+++

Is your application delivered as a set of services running on top of Linux? Did you think about writing a custom controller service that would start your application services in the correct order and monitor their health? Please, stop thinking about it! In this blog post I would like to convince you that you can leverage the existing *systemd* service manager to control your application services to your greatest benefit.

<!-- more -->

[systemd](https://www.freedesktop.org/wiki/Software/systemd/) is the default service manager on all major Linux distributions. We're going to demonstrate how it can be used to control a custom multi-service application.

{% img right /images/posts/systemd-logo.png 200 300 %}

Our application consists of three services: Service 1, Service 2 and Service 3. The following set of requirements should be met when controlling the services using systemd:

1. User can permanently enable/disable any of the services
2. User can start and stop the services as a group
3. User can start and stop each service independently
4. Service 2 should start before the Service 3 can start
5. Service 3 should be stopped before the Service 2 can be stopped
6. Services 1 and 2 should start in parallel to speed up the application startup
7. Components should be monitored and restarted in the case of failure

Required service startup order:

{% img /images/posts/systemd_startup_dependencies.svg 300 400 Service startup order %}

## Creating the systemd unit files

Let's begin creating the systemd unit files. First, we'll define a pseudo-service called `app`. This service doesn't run any deamon. Instead, it will allow us to start/stop the three application services at once.

{% codeblock lang:ini app.service %}
[Unit]
Description=Application

[Service]
# The dummy program will exit
Type=oneshot
# Execute a dummy program
ExecStart=/bin/true
# This service shall be considered active after start
RemainAfterExit=yes

[Install]
# Components of this application should be started at boot time
WantedBy=multi-user.target
{% endcodeblock %}

In the next step, we'll create systemd unit files for the three services that constitute our application. I included some explanatory comments in the first `app-component1` service definition. The definitions of the remaining two services `app-component2` and `app-component3` follow the same schema.

{% codeblock lang:ini app-component1.service %}
[Unit]
Description=Application Component 1
# When systemd stops or restarts the app.service, the action is propagated to this unit
PartOf=app.service
# Start this unit after the app.service start
After=app.service

[Service]
# Pretend that the component is running
ExecStart=/bin/sleep infinity
# Restart the service on non-zero exit code when terminated by a signal other than SIGHUP, SIGINT, SIGTERM or SIGPIPE
Restart=on-failure

[Install]
# This unit should start when app.service is starting
WantedBy=app.service
{% endcodeblock %}

The definition of the service `app-component2` resembles the definition of service `app-component1`:

{% codeblock lang:ini app-component2.service %}
[Unit]
Description=Application Component 2
PartOf=app.service
After=app.service

[Service]
ExecStart=/bin/sleep infinity
Restart=on-failure

[Install]
WantedBy=app.service
{% endcodeblock %}

We would like the service `app-component3` to start after the service `app-component2`. Systemd provides the directive `After` to configure the start ordering. Note that we don't use a `Wants` directive to create a dependency of `app-component3` on `app-component2`. This `Wants` dependency would instruct systemd to start the `app-component2` whenever the `app-component3` should be started. The `app-component2` would be started even in the case that it was disabled before. This is however not what we wanted as we require the user to be able to permanently disable any of the components. If `app-component2` is not enabled, systemd should just skip it when starting the application services.

{% codeblock lang:ini app-component3.service %}
[Unit]
Description=Application Component 3
PartOf=app.service
After=app.service
# This unit should start after the app-component2 started
After=app-component2.service

[Service]
ExecStart=/bin/sleep infinity
Restart=on-failure

[Install]
WantedBy=app.service
{% endcodeblock %}

## Adding the application services to systemd

After we finished the creation of the four systemd unit files, we need to copy them to the systemd configuration directory:

{% codeblock lang:sh %}
$ sudo cp app-component1.service app-component2.service app-component3.service app.service /etc/systemd/system/
{% endcodeblock %}

Next, we have to ask systemd to reload its configuration:
{% codeblock lang:sh %}
$ sudo systemctl daemon-reload
{% endcodeblock %}

If everything went well, you should be able to see the new services in the unit file list:

{% codeblock lang:sh %}
$ systemctl list-unit-files app*
UNIT FILE              STATE
app-component1.service disabled
app-component2.service disabled
app-component3.service disabled
app.service            disabled

4 unit files listed.
{% endcodeblock %}


## Testing the application services

After all the hard work we're now ready to exercise our configuration. First, let's enable all the application services:

{% codeblock lang:sh %}
$ sudo systemctl enable app app-component1 app-component2 app-component3
Created symlink from /etc/systemd/system/multi-user.target.wants/app.service to /etc/systemd/system/app.service.
Created symlink from /etc/systemd/system/app.service.wants/app-component1.service to /etc/systemd/system/app-component1.service.
Created symlink from /etc/systemd/system/app.service.wants/app-component2.service to /etc/systemd/system/app-component2.service.
Created symlink from /etc/systemd/system/app.service.wants/app-component3.service to /etc/systemd/system/app-component3.service.
{% endcodeblock %}

To start all the services, you can type:

{% codeblock lang:sh %}
$ sudo systemctl start app
{% endcodeblock %}

The systemd status command should display all the services as `active`:

{% codeblock lang:sh %}
$ sudo systemctl status app*
{% endcodeblock %}

Note that while the component services are marked as `running`, the pseudo-service `app` is showed as `exited`. This is an expected behaviour as the service `app` is of type `oneshot`. (I'm not showing the output of the systemctl status command here).

Next, let's check the starting order of the services. Systemd logs its messages into the journal. Let's list the recent log entries with:

{% codeblock lang:sh %}
$ journalctl -e
{% endcodeblock %}

In our sample output we can see that the service `app-component3` was indeed started after the service `app-component2`:

{% codeblock %}
Dec 05 20:34:57 localhost.localdomain systemd[1]: Starting Application...
Dec 05 20:34:57 localhost.localdomain systemd[1]: Started Application.
Dec 05 20:34:57 localhost.localdomain systemd[1]: Started Application Component 1.
Dec 05 20:34:57 localhost.localdomain systemd[1]: Starting Application Component 1...
Dec 05 20:34:57 localhost.localdomain systemd[1]: Started Application Component 2.
Dec 05 20:34:57 localhost.localdomain systemd[1]: Starting Application Component 2...
Dec 05 20:34:57 localhost.localdomain systemd[1]: Started Application Component 3.
Dec 05 20:34:57 localhost.localdomain systemd[1]: Starting Application Component 3...
{% endcodeblock %}

Now we're going to check that we can disable any service independently of other services. Let's stop the service `app-component2` and disable it:

{% codeblock lang:sh %}
$ sudo systemctl stop app-component2
$ sudo systemctl disable app-component2
Removed symlink /etc/systemd/system/app.service.wants/app-component2.service.
{% endcodeblock %}

When we stop and start our application the service `app-component2` will remain disabled.

{% codeblock lang:sh %}
$ sudo systemctl stop app
$ sudo systemctl start app
{% endcodeblock %}

Unfortunately, I realized that when using the `restart` command, systemd will enable the service `app-component2`:

{% codeblock lang:sh %}
$ sudo systemctl restart app
{% endcodeblock %}

The behaviour of the systemd `stop` command followed by the `start` command is not consistent with the behaviour of the systemd `restart` command. As a workaround, you can use the systemd `mask` command to really disable the application service, for example:

{% codeblock lang:sh %}
$ sudo systemctl mask app-component2
{% endcodeblock %}

This way, the service `app-component2` remains disabled no matter what.

## Conclusion

systemd provides a feature-rich service manager. Instead of implementing a home-grown solution you might want to think about using systemd to control your application services. Some of the benefits of opting for systemd are:

1. systemd is a standard service manager any Linux administrator is familiar with
2. systemd is battle tested to the extent your proprietary solution probably cannot be
3. You can discover many other useful systemd features which you can employ right away

Have fun!
