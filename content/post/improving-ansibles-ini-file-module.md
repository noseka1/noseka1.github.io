+++
title = "Improving Ansible's ini_file Module"
date = "2015-08-03"
slug = "2015/08/03/improving-ansibles-ini-file-module"
Categories = [ "devops" ]
+++

**Update 3/31/2016:** The implementation of the ini_file module described in this blogpost has been merged into Ansible version 2.0.

For editing Windows INI files, Ansible comes with an `ini_file` module built in. Unfortunately, this module uses Python's `ConfigParser` module which reformats the entire INI file whenever you want to change a single line. It removes all the comment lines, too. For me this was not acceptable. After looking for a possible solution I decided to improve the `ini_file` module and created `ini_file2`. I realized how easy it is to create an Ansible module.

<!--more-->

On Debian Linux, the Ansible's built-in `ini_file` module can be found at `/usr/share/ansible/files/ini_file`. This file is the base for our own `ini_file2`. The question was, at what location should one store the `ini_file2` module for Ansible to find it? From Ansible's [documentation](http://docs.ansible.com/ansible/developing_modules.html "Developing Modules") I learned that when looking for modules, Ansible searches the `./library` directory alongside of the top level playbooks. That sounds perfect to me.

After a while working with the Python code, I created the `ini_file2` module. This module provides an equivalent functionality to the original `ini_file` module, however, it does only the minimum changes when editing the INI file. It typically modifies only one line. When removing options, it doesn't delete the lines but comment them out instead. If there was a commented out option it comments it in when required.

## The ini_file and ini_file2 comparison

Let's compare the `ini_file` and `ini_file2` on a practical example. Our input INI file looks as follows:

{{< highlight ini "linenos=table" >}}
# This is the main configuration section
# There are some important options to configure
[main]
# This is the first option
option1 = orig_value

# This is the second option
# If not set, the default value is def_value
# option2 = def_value

# This is the last option
# If not set, the default value is def_value
option3 = some_value
{{< / highlight >}}

The Ansible test script will set the `option1` and `option2` to `new_value` and it will remove the `option3` from the INI file:

{{< highlight yaml "linenos=table" >}}
- name: Set option1 to new_value
  ini_file: dest=settings.ini section=main option=option1 value=new_value

- name: Set option2 to new_value
  ini_file: dest=settings.ini section=main option=option2 value=new_value

- name: Remove option3
  ini_file: dest=settings.ini section=main option=option3 state=absent
{{< / highlight >}}

When using the original `ini_file` module, the resulting INI file looks like this:

{{< highlight ini "linenos=table" >}}
[main]
option1 = new_value
option2 = new_value

{{< / highlight >}}

You can see that there's not much left from the input file. All comments are gone. In contrast, the `ini_file2` module does the editing operations with more precision:

{{< highlight ini "linenos=table" >}}
# This is the main configuration section
# There are some important options to configure
[main]
# This is the first option
option1 = new_value

# This is the second option
# If not set, the default value is def_value
option2 = new_value

# This is the last option
# If not set, the default value is def_value
#option3 = some_value
{{< / highlight >}}

## References

The `ini_file2` source code as well as test scripts can be found at [GitHub](https://github.com/noseka1/ini_file2 "ini_file2").
