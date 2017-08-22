---
layout: post
title: "Centralized Configuration Management, Our Approach"
date: 2017-08-20 13:09:28 -0700
comments: true
categories: development
---

Are you migrating your existing application to the cloud? Are you missing a solution for the centralized configuration management? Read on to learn, how we implemented the centralized configuration management on top of our existing application and got it ready for the cloud.

<!-- more -->

Our existing application is a [SOA-based](https://en.wikipedia.org/wiki/Service-oriented_architecture) application, i.e an application consisting of multiple services that communicate with each other and that are deployed across multiple nodes. Services are configured using configuration files located on the local filesystem. Currently, the operator has to edit multiple configuration files across multiple nodes by hand. Besides that, for the sake of performance and redundancy, there are multiple instances of a each service deployed behind a load balancer. This increases the configuration burden even further as the operator has to keep the configuration files consistent across several instances of the same service.

With the growing number of services and the need to deploy our application into the dynamic cloud environments, a centralized configuration management became a necessity.

## Looking for a solution

It would be possible to leverage the standard DevOps tools like Puppet, Chef or Ansible to manage the configuration files on each of the deployed nodes. For bare metal deployments or when deploying on virtual machines in the cloud, these tools could do a decent job. However, on our way to the cloud, we’re looking at containerizing all of our services. Furthermore, down the route we would like to leverage the serverless architecture, too. For updating a handful of configuration files inside of a Docker container, Puppet, Chef or Ansible just seem too heavy. Needless to say that these tools would not be usable when considering the serverless architecture.

When searching for a solution, we came across the [confd](https://github.com/kelseyhightower/confd) and [consul-template](https://github.com/hashicorp/consul-template) projects. Both tools are based on the same principle. First, the values of the configuration options are persisted in a backend store. While consul-template can store values in Consul only, confd supports a host of different backends like Consul, etcd, Redis or DynamoDB. Second, a set of templates on the filesystem is populated with the values from the backend store and hence forming valid configuration files. These configuration files are then consumed by the application. We drew a great deal of inspiration from this approach.

## Our approach

Our centralized configuration management consists of two components: *CCS* (Centralized Configuration Store) which is a Consul cluster holding the configuration data, and *CCT* (Centralized Configuration Tool) which is a command-line client. CCT implements two functions. First, it allows the operator to query and modify the configuration values persisted in CCS. Second, it syncs up the configuration files on the local filesystem with their state in CCS. The following diagram depicts the components involved in the centralized configuration management:

{% img left /images/posts/centralized_configuration_management.png %}

Besides the configuration values, the CCS component also stores all the additional data that is needed to completely recreate a given configuration file on the local filesystem. For instance, in the case of an ini file, CCS stores the absolute file path, file owner, file group, file mode, sections of the ini file, ini options with their values and all comment lines. Each ini option is also assigned a type or a set of allowed values and any configuration changes made by the operator are checked against the type information before they are accepted.

Apart from the ini file format, Java properties and XML files are also supported. The configuration management verifies that the XML file either conforms to a specific XML schema or is well-formed, before it is accepted. Lastly, all other configuration files that are not parsed by the configuration management are marked as "unmanaged" and the entire content of such a file is stored under a single key in the key-value store in Consul.

Next, let's review an example scenario where an operator wants to modify a configuration of a specific service. The individual steps are depicted in the diagram above:

1.  Using the CCT command-line client, the operator obtains a list of configuration files for a specific service managed by CCS. The operator uses CCT to edit the selected configuration file. CCT fetches all the data from CCS that are required to recreate the configuration file and present it to the operator for editing (for example by opening the file in the operator's favorite editor).

2.  After the operator made changes to the configuration file, the CCT parses the file to find out which values have been modified. The modified values are checked for corectness by CCT before they are saved in CCS.

3. Upon request, CCT fetches the configuration data from CCS in order to use it in the next step.

4. CCT syncs up the configuration files on the local filesystem with the data fetched from CCS.

5. The new configuration takes effect after the respective service has been restarted by the operator.

Overall, CCS is a single source of truth for the application configuration. This is in contrast with the confd or consul-template approach where the configuration values are stored in the backend while the templates are stored on the filesystem. When a new release of the application is deployed, having all configuration data in one place makes the upgrade of the configuration data easier.

## Future directions

It was important to us to introduce the centralized configuration management into our existing application without breaking the existing operational workflows. For example, operators should be able to edit the configuration files as they did in the previous versions of our application. Also, as the configuration files are written to the filesystem, the existing services continue to work without any modification from the previous versions. Hence the centralized configuration management can be deployed as a truly optional component on top of the existing application.

In the future, when some of our services will be deployed as (Lambda) functions in the serverless environment, those services will need to fetch their configuration by directly contacting CCS. However, nothing will change from the operator's standpoint. The operator will continue editing configuration files even when those won't exist on any filesystem anymore.

Do you use confd or consul-template to configure your application? Or did you build your own centralized configuration management? I would like to hear your comments. Feel free to use the comment section below.
