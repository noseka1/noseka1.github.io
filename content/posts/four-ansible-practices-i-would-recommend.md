+++
title = "Four Ansible Practices I Would Recommend"
date = "2018-06-17"
slug = "2018/06/17/four-ansible-practices-i-would-recommend"
Categories = []
+++

[Ansible](https://www.ansible.com/) is a popular IT automation tool. In this article, I would like to share with you some of the useful practices we have discovered while using Ansible over the past four years.

<!-- more -->

## Create a separate account for Ansible

{% img right /images/posts/ansible_logo.png 150 150 %}

In order to apply configuration changes, Ansible connects to the target machine via SSH. We prefer to create a dedicated `ansible` account on each machine managed by Ansible. This account is set up with the required `sudo` privileges. Password authentication to the `ansible` account is disabled. Instead, access is managed by adding or removing person's SSH public key to the `ansible` user's `authorized_keys` file.

SSH daemon logs the SSH key fingerprint that was used for authentication. That allows us to keep track of who made use of the `ansible` account. On Red Hat based distros, you can find the access logs in `/var/log/secure`. Here is a sample log message after I triggered the execution of an Ansible playbook:

```
2018-06-16T19:33:02.298905-07:00 machine1 sshd[29892]: Accepted publickey for ansible from 10.0.0.253 port 54600 ssh2: RSA f4:83:34:8f:f8:7d:29:0e:40:65:b9:bc:a0:bb:eb:d0
```

Furthermore, any time Ansible uses `sudo` to execute a task, an additional log message is appended to the `/var/log/secure` log file on the target machine. We found that for the auditing purposes the generated logs are sufficient.

In the cloud, we either use custom images that already contain the `ansible` account or we create the `ansible` account on the first boot using a cloud-init script. Using a dedicated `ansible` account instead of the `ec2-user`, `cloud-user`, `centos` and other image accounts streamlines our management efforts.

## Report changes made on the target machine accurately

When Ansible executes a playbook, it highlights all configuration changes made to the target machine while bringing it to the desired state described in the playbook. Administrator can review the reported changes to verify that they meet his expectations.  If no changes had to be made to the target machine, Ansible should report `changed=0` after the playbook execution completes.

When executing Ansible `command` module or `shell` module, there's no good way for Ansible to know whether the executed command or shell script made any changes to the target machine. Here we can help ourselves with a neat trick that I will explain on the following example.  Imagine that we want to create a new InfluxDB database:

{% codeblock %}
- name: Create a database in InfluxDB
  shell: |
    set -e
    if ! influx -execute 'show databases' | grep {% raw %}{{ database_name }}{% endraw %}; then
      influx -execute 'create database {% raw %}{{ database_name }}{% endraw %}'
    fi
{% endcodeblock %}

The above task creates a new InfluxDB database in the case that it doesn't already exist. The problem with the above code is that Ansible will always mark this task as changed. In order to report the change status accurately, let's leverage Ansible's `changed_when` directive:

{% codeblock %}
- name: Create a database in InfluxDB
  shell: |
    set -e
    if ! influx -execute 'show databases' | grep {% raw %}{{ database_name }}{% endraw %}; then
      influx -execute 'create database {% raw %}{{ database_name }}{% endraw %}'
      echo CHANGED
    fi
  register: create_database
  changed_when: create_database.stdout | search("CHANGED")
{% endcodeblock %}

We modified the shell script to print `CHANGED` to the standard output if and only if a new InfluxDB database has been created. Next, we captured the entire standard output of the shell script in the `create_database` variable using the Ansible's `register` directive. Lastly, we search the content of the `create_database` variable for the `CHANGED` keyword. If the `CHANGED` keyword is found, Ansible will mark the task as changed.

It would be great if Ansible would come up with a mechanism to determine whether a shell script made changes to the target machine or not. In the meanwhile, the above pattern works very well for us.

## Skip completed tasks

Ansible playbook may be executed multiple times against the same target machine. During each run Ansible must be able to determine which tasks should be executed and which tasks should be skipped because they were already completed in the previous run. Think about the scenario where we are configuring a Docker storage. In the following code, we are invoking the `docker-storage-setup` command on the target machine:

{% codeblock %}
- name: Run the Docker storage configurator
  command: docker-storage-setup
  become: yes
{% endcodeblock %}

If you would run the above command the second time, it would fail because the Docker storage has already been configured. How to ensure that the above task will run only once? Ansible's `command` module provides `creates` and `removes` parameters that instruct Ansible to only run the `command` task if a specific file does or doesn't exist on the file system. The idea is that the executed command creates or deletes these files. Could these parameters help us here? While these parameters can save you in many cases, we would like to avoid using them in this situation. Firstly, we would have to go and figure out whether the `docker-storage-setup` script creates or removes any file. Secondly, the location of the created or removed file may depend on the chosen storage driver which would demand extending our script with additional logic. And lastly, there is no guarantee that the future versions of the `docker-storage-setup` script would manipulate the same file which could cause our Ansible script to break. As an alternative, here is a pattern that a software practitioner can use in such a situation:

{% codeblock %}
- name: Check the docker_storage_initialized stamp
  stat: path=docker_storage_initialized
  register: storage_initialized

- block:
    - name: Run the Docker storage configurator
      command: docker-storage-setup
      become: yes

    - name: Create the docker_storage_initialized timestamp
      file: path=docker_storage_initialized state=touch

  when: not storage_initialized.stat.exists
{% endcodeblock %}

The gist of the above code is simple. After we have successfully configured the Docker storage we create a stamp file. Because the stamp file exists, Ansible will skip the Docker storage configuration in all the following playbook runs. The stamp file is created in a well defined location. Should we want to re-run the Docker storage configuration in the future, we can just remove the stamp file before executing the playbook.

## Backporting Ansible modules

For the task at hand, it is always preferable to leverage a dedicated Ansible module before falling back on generic `command` or `shell` modules. In our Ansible practice, we came into situations where the Ansible module we would like to use was available only in the newer versions of Ansible or that the module in our version of Ansible was buggy. As the interface between Ansible core and the Ansible modules is pretty stable, here is my advice: just copy the desired module from the newer version of Ansible and drop it into the `library` directory next to your playbook. This `library` directory is a location where you can put your custom modules. Moreover, any custom module having the same name will override a module distributed with Ansible. This is how our `library` directory currently looks like:

{% codeblock lang:sh %}
$ ls -1 library
nsupdate.py
win_get_url.ps1
win_get_url.py
win_reg_stat.ps1
win_reg_stat.py
{% endcodeblock %}

We are using Ansible version 2.2.1.0 which doesn't contain `nsupdate` and `win_reg_stat` modules. Also, we found the `win_get_url` module in version 2.2.1.0 to be buggy. Adding a handful of modules to the `library` directory avoids the need to upgrade to the next version of Ansible. From past experience we know that porting our Ansible code base to the next version of Ansible requires a considerable effort.

## Conclusion

In this article, we shared some of the Ansible practices we found useful. We recommended to create a dedicated `ansible` account on each machine that is managed by Ansible. We described how to inform Ansible whether a script did or did not make any changes to the target machine. We showed you how to prevent a repeated execution of Ansible tasks by creating a custom stamp file. Lastly, we demonstrated that backporting Ansible modules is not that complicated as it may sound.

Hope you enjoyed this article. Let me know if you have any comments or suggestions to the presented practices. Feel free to add your comments in the comment section below.
