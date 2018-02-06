---
layout: post
title: "How I Became a Red Hat Certified Architect in One Year"
date: 2017-08-01 22:47:14 -0700
comments: true
categories: cloud
---

Roughly a year ago, my boss offered to me a Red Hat Learning Subscription. Because continuous education belongs to the habits of a good software practitioner, I appreciated this opportunity to deepen my knowledge of Red Hat technologies. At that time I didn't have an idea about how much fun I was going to have on my journey to become a Red Hat Certified Architect. Read more, if you want to find out!

<!-- more -->

In Wikipedia, you can find a great [overview](https://en.wikipedia.org/wiki/Red_Hat_Certification_Program) of the Red Hat certification program. To achieve the Red Hat Certified Architect level, you have to pass seven exams in total. The certification path starts with the RHCSA and RHCE exams. After becoming RHCE, you will have to pass five more exams based on your choice to obtain the RHCA certificate.

## Starting off with RHCSA and RHCE

{% img right /images/posts/rhcsa_logo.png 130 130 %}

The certification path begins with an entry-level certification - Red Hat Certified System Administrator ([RHCSA](https://www.redhat.com/en/services/certification/rhcsa)). In the exam, I had to demonstrate basic Linux administration skills like creating users and groups, managing file system permissions including POSIX access control lists (ACLs), creating cron jobs, basics of SELinux, managing software packages with yum, creating and mounting local file systems, working with LVM, network configuration, mounting NFS and SMB file systems and firewall configuration. I had plenty of time to complete all the exam tasks. Overall, RHCSA didn't seem too difficult to me.

{% img right /images/posts/rhce_logo.png 130 130 %}

About two weeks after passing the RHCSA, I took the next exam - the Red Hat Certified Engineer ([RHCE](https://www.redhat.com/en/services/certification/rhce)) exam. The main focus of this exam was configuration of a caching DNS server (unbound), SMTP nullclient (Postfix), configuration of an iSCSI target and initiator, configuration of Apache web server including HTTPS, running MariaDB, configuration of NFS and SMB servers and basic shell scripting. Based on the experience from the RHCSA exam, I thought that I would have more than enough of time again to complete my tasks and perhaps make myself a coffee, too. How wrong I was! The RHCE exam is loaded with so many tasks that you will be very busy for the entire 3.5 hours. Due to my rather slow and relaxed approach at the beginning of the exam, I was not able to complete all the tasks in time. In the end, I was very happy that I still passed.

You can find further details about my RHCSA/RHCE experience in [this](/blog/2016/11/07/rhcsa-slash-rhce-exam-experience/) blog post.

## Climbing to the top

{% img right /images/posts/rhca_logo.png 130 130 %}

After becoming a Red Hat Certified Engineer, I turned my attention to the Red Hat Certified Architect ([RHCA](https://www.redhat.com/en/services/certification/rhca)) certification. To achieve this highest level of certification, I had to pass five additional exams based on my own selection. Red Hat provides about twenty exams to choose from, divided into concentrations like Datacenter, DevOps, Application platform or Cloud. The concentrations are only advisory. You can pick exams across concentrations which I also did.

At the time I was choosing my next exam, I was already working with OpenShift for several months. In order to increase my knowledge of OpenShift, I decided to take the OpenShift certification exam next. You can find my experience from the Red Hat Certificate of Expertise in Platform-as-a-Service ([EX280](https://www.redhat.com/en/services/training/ex280-red-hat-certificate-expertise-platform-service-exam)) exam in [this](/blog/2017/04/04/passed-the-openshift-ex280-certification/) blog post. I passed this exam with my lowest score ever but yeah, I did pass.

During the preparation for the OpenShift exam, I had to work with Docker containers rather intensively. It would make sense to continue down the container route and take the container exams [EX270](https://www.redhat.com/en/services/training/ex270-red-hat-certificate-expertise-atomic-host-container-administration) and [EX276](https://www.redhat.com/en/services/training/ex276-red-hat-certificate-expertise-containerized-application-development) next. However, I realized that the OpenStack certification exams which I planned to take, too, were not available at my location. As I already had travel plans to Prague where the OpenStack exams were available, I decided to shift the gears and started preparing for the two OpenStack exams Red Hat Certified System Administrator in Red Hat OpenStack ([EX210](https://www.redhat.com/en/services/training/ex210-red-hat-certified-system-administrator-red-hat-openstack-exam)) and Red Hat Certified Engineer in Red Hat OpenStack ([EX310](https://www.redhat.com/en/services/training/ex310-red-hat-certified-engineer-red-hat-openstack-exam)). I was able to pass these two exams with the maximum score of 300 points as you can read in [this](/blog/2017/06/26/acing-the-red-hat-openstack-certification-exams/) article.

The last two exams on my journey were the Red Hat Certificate of Expertise in Atomic Host Container Administration ([EX270](https://www.redhat.com/en/services/training/ex270-red-hat-certificate-expertise-atomic-host-container-administration)) and the Red Hat Certificate of Expertise in Containerized Application Development ([EX276](https://www.redhat.com/en/services/training/ex276-red-hat-certificate-expertise-containerized-application-development)). I wrote about my experiences from these two exams [here](/blog/2017/07/29/passed-red-hat-container-certifications-ex270-and-ex276/). These two exams concluded my journey and I became a Red Hat Certified Architect.

Red Hat maintains a web page showing a track record of each Red Hat Certified Professional. The list of my Red Hat certifications can be found on this [Verify a Red Hat Certified Professional](https://www.redhat.com/rhtapps/certification/verify/?certId=160-216-727) page.

## What's next?

The RHCA status doesn't last forever as I learned on the Red Hat web page describing how to [stay current](http://servicesblog.redhat.com/2016/09/23/stay-current/). In order for my RHCA to stay current, I must maintain five eligible credentials beyond the RHCE level. Certificates that I obtained beyond the RHCE are valid for three years. That means that I must take five exams every three years in order to maintain my RHCA status. Those five exams can be the same five exams I already passed before or I can pick any other exams from the selection of exams eligible for RHCA. Besides that, as a RHCA I'm entitled to a 50 percent discount on the [Red Hat Learning Subscription - Standard](https://www.redhat.com/en/about/videos/red-hat-learning-subscription-standard). This subscription includes all the prep materials and five first exam attempts.

It seems to me that the best way to maintain my RHCA status would be to buy the Learning Subscription every three years and use it to pass five exams. Currently, the Learning Subscription costs $7000 which would be $3500 after the 50 percent discount. In three years from now, I'm looking forward to the respective conversation with my boss ;-)

Are you a fellow RHCA? Would you like to share some details about your certification journey? And do you plan do maintain your RHCA? I would like to hear from you, feel free to leave your comments below.
