+++
title = "Five Basic Tips for Teamwork With Git"
date = "2015-04-18"
slug = "2015/04/18/five-basic-tips-for-teamwork-with-git"
Categories = [ "development", "git" ]
+++
Do you care about how your Git commits look like? A great software practitioner does, indeed. Let's review a couple of basic tips for developers that make the Git commit log look good and teamwork with Git source control more fun.
<!--more-->

1) Introduce yourself to Git
----------------------------

When searching through the commit history your fellow developers would like to recognize that this particular awesome commit was created by you. Before your first commit, please, introduce yourself to Git. Tell Git your name and email address:

{{< highlight shell "linenos=table" >}}
$ git config --global user.name "Ales Nosek"
$ git config --global user.email "anosek@verimatrix.com"
{{< / highlight >}}

2) Commit message formatting
----------------------------

Respect the conventions for Git commit message formatting. The first line of the commit message is a short (50 chars or less) summary. The first letter of the summary is capitalized and there's no dot at the end of the summary. The summary begins with a reference to the issue in your bug tracking system if available. Use imperative in your summary line as you'd be commanding your code to do something, i.e. write "Remove obsolete code" instead of "Removed obsolete code" or "Removes obsolete code". More detailed explanatory text comes after a blank line. Here is a model Git commit message:

{{< highlight plaintext "linenos=table" >}}
PROJXREF-27 Remove obsolete code in component X

The logging functionality in component X is not needed anymore.
Component Y took over the logging responsibility.
{{< / highlight >}}

See also this [article](http://chris.beams.io/posts/git-commit/ "How to Write a Git Commit Message") for more detailed explanation and examples.

3) One commit per unit of work
------------------------------

Create one commit per unit of work. A combined commit like "Add method X, correct indention, clean up whitespaces" is harder to review. Break your changes down into multiple commits, e.g. three commits "Add method X", "Correct indention" and "Clean up whitespaces".

4) git diff `--`cached
-----------------------

Before commiting *always* check your code changes. Make sure that your commit includes only the changes you intended to commit. Let your debug code and test modifications not flow into the production code base. Before committing double-check your changes with:

{{< highlight shell "linenos=table" >}}
$ git diff --cached
{{< / highlight >}}

5) Trailing whitespaces
-----------------------

Whitespace changes in the commit diffs decrease the readability of the commit diffs and make code review less fun. You can configure your editor to remove the trailing whitespaces for you on file save. Perhaps a better option though is to instruct Git to clean up the trailing white spaces automatically before comitting. You can use the commit hook [here](http://stackoverflow.com/questions/591923/make-git-automatically-remove-trailing-whitespace-before-committing/3516525#3516525 "Make git automatically remove trailing whitespace before committing") to do exactly that. Save the commit hook as file named `pre-commit`. Install the `pre-commit` script into your Git repository:

{{< highlight shell "linenos=table" >}}
$ cp pre-commit my_existing_repo/.git/hooks
$ chmod 755 my_existing_repo/.git/hooks/pre-commit
{{< / highlight >}}

The pre-commit hook will run only before the commit in this particular Git repository. If you'd like Git to install the pre-commit hook to every newly created/cloned repository you'll need to add the hook file into your template directory. First tell Git where is your template directory located by adding the following into your `~/.gitconfig` file:

{{< highlight ini "linenos=table" >}}
[init]
        templatedir = ~/.gittemplate
{{< / highlight >}}

Now you can create your Git template directory and copy the `pre-commit` hook file into it:

{{< highlight shell "linenos=table" >}}
$ mkdir -p ~/.gittemplate/hooks
$ cp pre-commit ~/.gittemplate/hooks
$ chmod 755 ~/.gittemplate/hooks/pre-commit
{{< / highlight >}}

From now on whenever you create/clone a Git repository you should find the `pre-commit` hook file installed at `.git/hooks/pre-commit`. Git will clean up trailing whitespaces for you before you commit.
