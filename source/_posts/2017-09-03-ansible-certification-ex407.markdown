---
layout: post
title: "Ansible Certification EX407"
date: 2017-09-03 22:47:06 -0700
comments: true
categories: devops
---

Last week I passed the Red Hat certification exam [EX407 Red Hat Certificate of Expertise in Ansible Automation](https://www.redhat.com/en/services/training/ex407-red-hat-certificate-expertise-ansible-automation). In this blog post, I'd like to share some of my experience with you.

<!-- more -->

{% img right /images/posts/ansible_logo.png 80 80 %}

{% img right /images/posts/redhat_logo.png 130 130 %}

In addition to the Ansible certification, Red Hat offers a certification for Puppet as well, [EX405 Red Hat Certificate of Expertise in Configuration Management with Puppet](https://www.redhat.com/en/services/training/ex405-red-hat-certificate-expertise-configuration-management-puppet). I was using Puppet in the years 2008/2009. Back then, Puppet was a state of the art configuration management tool that did a great job for us. However, later on Ansible showed up and I quickly realized that the problems that we were solving with Puppet could have been more easily solved with Ansible. In our company, we switched from Puppet to Ansible in 2014 and have never looked back. As I gained much more experience using Ansible, I decided to go for the Ansible certification. For you, perhaps the Puppet certification would be more interesting.

It is the way of testing, that makes the Red Hat certification exams so enjoyable for me. Red Hat certification exams are purely practical. In the exam, you'll be given a list of requirements and an access to one or more virtual machines. Your goal is to configure the virtual machines based on the requirements.

The Ansible exam focused on writing Ansible playbooks, working with Ansible inventories including dynamic inventories, running ad-hoc Ansible commands, leveraging Ansible facts, creating Jinja2 templates, error handling, playbook tags, downloading a role from Ansible Galaxy and using Ansible vault to secure passwords.

To prepare for the exam, I used the online course Automation with Ansible I (DO407R) that is included in my [Red Hat Learning Subscription](https://www.redhat.com/en/services/training/learning-subscription).

Overall, I didn't find the exam too difficult. There were 4 hours of time provided for the exam. I was able to complete all my tasks 1 hour and 20 minutes before the exam end while reaching the maximum score of 300 points.

If you've kept reading my blog post until this point, here are three basic exam tips for you:

1.  Use the `ansible-playbook --limit` parameter, to try out your playbook against a single host first before applying it to all your hosts.
2.  Make sure you can run Ansible modules as ad-hoc commands. They can come handy when you want to undo the effects of running an erroneous playbook.
3.  Use `ansible-doc --list` to look up a module that can achieve your goal and `ansible-doc <module>` to learn about the module parameters and example usage.

Passing the Ansible exam brought me to the rank of Red Hat Certified Architect Level II. The current list of my Red Hat certifications can be found on the [Verify a Red Hat Certified Professional](https://www.redhat.com/rhtapps/certification/verify/?certId=160-216-727) website.

Are you working on your Red Hat certification and would like to share your experience? I would always like to hear from you! Feel free to leave your comments below.
