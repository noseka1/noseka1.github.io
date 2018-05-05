---
layout: post
title: "Designing a Common Build System"
date: 2018-05-03 21:57:40 -0700
comments: true
categories: development
---

Code reuse belongs to the basic tenets of software development. Moreover, one should have the same principle in mind when maintaining build scripts. If you are copy-pasting Makefiles and pom.xml files from project to project, stop now and read this article! We are going to discuss how to design a common build system.

<!-- more -->

A good software practitioner avoids having huge chunks of Makefiles, pom.xml files, build.xml files or shell scripts copy-pasted all over the code base. Based on experience, copy-pasted build scripts lead to inconsistencies, build issues and are overall driving the maintenance cost high.

In 2010, we invested heavily in the improvements of our build infrastructure. We introduced a continuous integration server Hudson (remember the project that was later renamed to Jenkins?), added source code analysis tool Sonar (nowadays called SonarQube), embraced RPM packaging and created a set of highly reusable build scripts which we called a common build system. In the next section, I'm going to discuss a high-level design and ideas behind the common build system.

## High-level overview

Our build system supports Java, C++ and C development. The core of the build system comprises of:

* [Apache Ant](https://ant.apache.org/) An old-timer between build tools. In current times, writing build scripts in XML is not sexy anymore, however, we like Ant for its simplicity and power. If you cannot express the required functionality using Ant tasks, you can always defer to using an embedded JavaScript. In our Ant scripts, you could find the `<script language="javascript"> ... </script>` element that embeds JavaScript in several places.
* [Apache Ivy](http://ant.apache.org/ivy/) Is a very flexible dependency manager that integrates with Apache Ant. While Ivy is predominantly used to manage Java jar files, we use it to manage C/C++ dependencies, too. For that, we zip up the C++ header files and push it along with the C++ shared libraries into the artifact repository.
* [Ant Script Library](https://github.com/tumbarumba/AntScriptLibrary) Writing Ant build scripts from scratch is time consuming. To avoid spending this effort, we embraced Ant Script Library (ASL) which is a collection of re-usable Ant scripts providing a number of pre-defined targets.
* [Boilermake](https://github.com/dmoulding/boilermake) Boilermake is a reusable GNU Make compatible Makefile. It uses a non-recursive strategy which avoids the many well-known pitfalls of recursive make, see also [Recursive Make Considered Harmful](http://aegis.sourceforge.net/auug97.pdf). We leverage Boilermake to compile C/C++ source code. An Ant `build.xml` wrapper script calls Boilermake when building a C/C++ module.

The following diagram illustrates the organization of our code base:

{% img /images/posts/common_build_system.svg 600 600 %}

Our code base is divided up into modules. A module contains either Java or C/C++ source code required to build a library or executable. The `build-common` module is special in that it doesn't contain any source code to compile. Instead, it acts as a container in which we store all our reusable build scripts. The build scripts of other modules import the definitions from the `build-common` module. Due to high code reuse, the build scripts of individual modules are rather concise. The Ant statement to import the common definitions from the `build-common` module looks as follows:

{% codeblock lang:sh %}
<import file="../../Build/build-common/module.xml" />
{% endcodeblock %}

Modules are organized into Git repositories according to the functionality they implement. For instance, modules of a specific product reside in its own Git repository. Furthermore, the `Platform` repository groups together modules that implement common libraries shared across several products.

All artifacts exported by the individual modules (jars, shared libraries, header files) are shared between the modules via the artifact repository. If additional information needs to be shared between modules, modules can publish Java properties files into the artifact repository which other modules can fetch and import. It was important to us to avoid any direct references between modules on the file system with the exception of the reference to the `build-common` module. These direct references between modules would be less obvious than passing artifacts and extra information via the artifact repository. We like to keep a good track of the dependencies between our software modules.

## Module directory structure

Apache Ant does not propose any particular directory structure. However, it is easier to work with modules which have a common structure. Ant Script Library embraces the [Standard Directory Layout](https://maven.apache.org/guides/introduction/introduction-to-the-standard-directory-layout.html) from Maven project and so we derive our directory structure from this standard. Here is our module directory structure in greater detail:

| Directory          | Purpose                          |
|:------------------ |:-------------------------------- |
| src/main/java      | Java application/library sources |
| src/main/c++       | C++ application/library sources  |
| src/main/c         | C application/library sources    |
| src/main/resources | Application/library resources    |
| src/main/scripts   | Application/library scripts      |
| src/test/java      | Java test sources                |
| src/test/c++       | C++ test sources                 |
| src/test/c         | C test sources                   |
| build.xml          | Ant build file                   |
| ivy.xml            | Ivy module descriptor            |
| README.md          | Module's README file             |
| target             | Build output directory           |

## Common build targets

We maintain a set of common Ant build targets which every module must implement:

| Target name               | Description                                                                |
|:--------------------------|:-------------------------------------------------------------------------- |
| default                   | Build artifacts, publish artifacts                                         |
| distribute                | Build artifacts, publish artifacts and create a distribution package       |
| all                       | Build and test artifacts, publish artifacts, create a distribution package |
| clean                     | Delete files generated during the build                                    |
| clean-dist                | Delete files generated during the RPM package build                        |
| clean-all                 | Delete all generated files                                                 |
| ci-default                | Called by CI server during the build job execution                         |
| report-sonar              | Do statical analysis and send reports to Sonar server                      |

These build targets constitute a well-known interface which allows developers to clean and build any module using the same Ant command. Furthermore, each module defines the `ci-default` target which is called by the Jenkins CI server when building the module. This allows us to have Jenkins build any of our modules by issuing these two commands:

{% codeblock lang:sh %}
ant clean-all
ant ci-default
{% endcodeblock %}

This two-command interface establishes a contract between modules and Jenkins and allows us to keep the Jenkins job definition fairly static. On the other hand, developers have full power to define what should happen during the build by implementing the module's  `ci-default` target.

Target `report-sonar` runs the SonarQube source code analysis tool, and sends the collected data to the SonarQube server. As the source code analysis takes some time to complete, we don't run it on every push to the Git repository. Instead, we scheduled a nightly Jenkins job that analyzes all the modules and uploads the collected data at once.

## Final remarks

It has been several years since we created the common build system and we have been improving it ever since. We added many features to support our development process. We have already discussed some of them. Here is a summary of the most important capabilities we implemented so far:

* Support for building Java and C/C++ code
* Support for multiple target platforms (RHEL5, RHEL6, RHEL7, Solaris 10)
* Packaging software as RPM, Solaris pkg, IzPack and Docker image
* Import of test data into Oracle database before running the unit tests
* Management of build jobs in Jenkins
* Source code analysis using SonarQube

In our company, the relentless improvement process never stops. The Ant + Ivy tools we leverage at the core of our build system are past their prime. So, what's next? I'm very excited about our current progress in gradually replacing Ant + Ivy with the more modern Gradle build tool.

I hope you enjoyed the tour through the design of the common build system. I would be interested to know how you promote code reuse of the build scripts in your company. It would be great to hear about your approach. Please, feel free to leave your comments in the comment section below.
