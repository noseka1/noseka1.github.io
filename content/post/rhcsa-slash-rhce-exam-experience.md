+++
title = "RHCSA/RHCE Exam Experience"
date = "2016-11-07"
slug = "2016/11/07/rhcsa-slash-rhce-exam-experience"
Categories = [ "devops" ]
+++

Today was the great day when I passed the RHCE certification exam. If you're thinking about getting a Linux certification or you're already working towards RHCSA/RHCE, this blog post is for you.

<!--more-->

The RHCSA/RHCE certification is a part of the [Red Hat Certification Program](https://en.wikipedia.org/wiki/Red_Hat_Certification_Program). You have to pass two practical exams, RHCSA (Red Hat Certified System Administrator) and RHCE (Red Hat Certified Engineer).

# Modus operandi

Both the RHCSA and RHCE exams are hands on. You will be presented with one or more RHEL7 virtual machines and a list of Linux configuration tasks. Your goal is to set up the virtual machines as described in the tasks. After your exam is finished, the Red Hat test automation will reboot the virtual machines and run the grading scripts that will compute your score. An hour or two after the exam end you will receive your exam results in the email.

The exam strictly tests your practical skills which makes it in my opinion very valuable. Several years ago I passed some of the exams from the Oracle Java certification path. The major part of the Java exams was based on the multiple choice questions. It was less fun for me to prepare for it and I think that a hands on exam would test the Java development skills much accurately.

If you're a hiring manager searching for the Linux talent, you should know that:

>Job candidates that passed the RHCSA/RHCE certification really demonstrated their Linux administration skills at some point.

# Tips

You can find a ton of tips for the RHCSA/RHCE exams on the Internet already. I'd like to share some more advice here.

The RHEL7 virtual machines you will be working on have manual pages installed. Before taking the exam, make sure you know where to look for the useful configuration examples. For instance, when configuring the network interfaces you can refer to:

{{< highlight shell "linenos=table" >}}
$ man 5 nmcli-examples
{{< / highlight >}}

Full examples of commands to configure the firewall can be found at:

{{< highlight shell "linenos=table" >}}
$ man 5 firewalld.richlanguage
{{< / highlight >}}

When wrestling with the Apache server configuration you can leverage the Apache manual. After you installed the `httpd-manual` RPM package, you can browse the manual using the *elinks* web browser:

{{< highlight shell "linenos=table" >}}
$ yum install elinks httpd-manual
$ elinks /usr/share/httpd/manual/index.html
{{< / highlight >}}

If it is feasible for you, make yourself a favor and buy the [Red Hat Learning Subscription](https://www.redhat.com/en/services/training/learning-subscription). You will get access to the training materials that walk you through the tasks very similar to what you'll see on the real exam. It includes mock exams. And you'll be provided with the virtual machines that you can practice on over and over again until you're ready to take the test.

I found the Red Hat learning materials to be of a high quality. Besides the prep for the exam, I'm using them to learn about further technologies from the Red Hat portfolio.

# Is the RHCSA/RHCE for me?

In my opinion, the RHCSA/RHCE is one of the best Linux certifications available. If you mean it with Linux seriously - and it doesn't necessarily have to be the Red Hat flavor of Linux - go get certified!
