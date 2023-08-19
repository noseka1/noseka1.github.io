+++
title = "Git Getting the Timing Right"
date = "2017-01-02"
slug = "2017/01/02/git-getting-the-timing-right"
Categories = []
+++

Do you work on a development team that is distributed across several time zones? Got confused by the dates that Git shows in the commit logs? In this blog post we're going to review some basics about how Git deals with time.

<!--more-->

## Pay attention to the commit timestamps

Let's assume that two developers work in the same Git repository. The first developer Prasad is located in Bangalore, India. His colleague Joe is located in San Diego, US. The Git log they created looks as follows:

{% codeblock lang:sh %}
$ git log
commit 0f7a87b545d23870aa3dfe2665e242fd1a807445
Author: Joe Smith <jsmith@sandiego.us>
Date:   Tue Dec 6 11:41:44 2016 -0800

    Commit 3

commit 62d95dfbc185ad60cd3ce2a4a3d02a12c1fb7dea
Author: Prasad Gupta <pgupta@bangalore.in>
Date:   Tue Dec 6 21:45:51 2016 +0530

    Commit 2

commit 106db71e39e3b25fb3fa3df4c55cc8f063e78ff5
Author: Prasad Gupta <pgupta@bangalore.in>
Date:   Tue Dec 6 21:45:00 2016 +0530

    Commit 1

{% endcodeblock %}

Git orders the commits based on the commit timestamps with the most recent commit shown on top. In the sample logs above you can see that all three commits were made on the same day Tuesday, December 6 2016. What might look a little bit odd is the order of the commits in the logs. `Commit 3` made at 11:41 should have actually appeared below the commits `Commit 1` and `Commit 2` made at 21:45, right? Wrong!

In the commit logs, Git displays the timestamp in the format *localtime + timezone offset*. When reading the timestamps it's important to take the timezone offset into account. Using the timezone offset you can convert the timestamp of the `Commit 2` and `Commit 3` into the UTC time in order to compare them. Because the `Commit 2` was made at 21:45 +0530 (= 16:15 UTC) and `Commit 3` was made at 11:41 -0800 (= 19:41 UTC) the `Commit 3` was created after the commits `Commit 1` and `Commit 2` and the chronological order displayed by Git is indeed correct.

## Check the time settings on the developer machines

The timestamp recorded in the Git commit is based solely on the current time on the machine where the commit was created. Even if you have a corporate Git server where you push all your commits to you have to know that the Git server doesn't modify the timestamps in any way. You have to encourage your developers to have the time on their machines set correctly. This includes the correct local time as well as the time zone. On the Linux machines equipped with [systemd](https://www.freedesktop.org/wiki/Software/systemd/) these time settings can be changed using the `timedatectl` command. Use `date` command to validate your settings:

{% codeblock lang:sh %}
$ date
Mon Jan  2 20:33:04 PST 2017
{% endcodeblock %}
The output shows my correct local time and the correct time zone (PST) as I'm located on the west coast of the US.

## Author date and commit date

In the aforementioned example with the `git log` command I simplified the situation a little bit. There are actually two different timestamps recorded by Git for each commit: the *author date* and the *commit date*. When the commit is created both the timestamps are set to the current time of the machine where the commit was made. The author date denotes the time when the commit was originally made and it never changes. The commit date is updated every time the commit is being modified for example when rebasing or cherry-picking.

By default `git log` orders the logs according to the commit date, however, the author date is actually displayed in the output. This can easily lead to confusion when there are commits present for which the commit date and the author date actually differ. The parameter `--author-date-order` can be used to order the commits based on the author timestamp:

{% codeblock lang:sh %}
$ git log --author-date-order
{% endcodeblock %}
