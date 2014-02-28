---
layout: post
title: "Enforcing unique cookbook versions"
date: 2014-02-27 14:04:18 -0500
comments: true
categories: 
- Chef
- IAC
- Versioning
- Spork
- Knife
- Jenkins
---

Infrastructure as Code (IAC) development projects need to enforce strict cookbook versioning.

Here is a diagram of a IAC workflow that uses Jenkins, Spork , Git and Knife to ensure only unique cookbook versions are uploaded to the Chef Server:

<img src="{{ root_url }}/images/IAC_Pipeline_Versioning.jpg" />

I encountered four unexpected issues while implementing this workflow.  

Below are the descriptions of these issues along with an explanation of their resolution.

First however, a big thanks to Tony Spring (http://www.porkchopsandpaintchips.com).  
The above workflow is based on conversations we had about a pipeline he has put into place. 


<h2> Issue #1 – Using “spork check” in a CI pipeline: </h2>

There was no option available to automatically bypass the “Y/N” prompt when “spork check” determined that a version bump was needed.  

The solution was to write a patch which adds a new "-p/--autobump" option to the spork check command. This new option allows a ci workflow to automatically bump cookbook versions when necessary.  

This patch was submitted as a pull request, and it has been accepted for the 1.3.3 Spork release. 

Until the 1.3.3 gem is available, the code can be cloned from github and compiled locally.  For more details, see: https://github.com/jonlives/knife-spork/pull/108

Using the updated gem, the CI pipeline can handle the version check and bump with a single command: 

{% codeblock lang:bash %}
knife spork check $COOKBOOK -o $WORKSPACE –p
{% endcodeblock %}

<h2> Issue #2 – Jenkins checks out Git projects with a detached HEAD </h2>

Jenkins checks out Git commits, not branches.  So the the working directory in the Jenkins job has a detached HEAD. 

One work around for this is to use the Git Plugin’s “Additional Behavors>Check out to specific local branch” option and define “master” (or any other name you want to use) as the local branch.  

Alternatively, the branch can be set from the Jenkins execute shell with a command like this:

{% codeblock lang:bash %}
git checkout master
{% endcodeblock %}

<h2> Issue #3 – tricky bug with git diff –quiet </h2>

Ultimately determining if spork has updated the Git working directory is as simple as running:

{% codeblock lang:bash %}
git diff --exit-code
GIT_STATUS=$?
{% endcodeblock %}

However, I spent a significant amount of time trying to suppress the diff output and therefore kept running into this bug:  http://git.661346.n2.nabble.com/bug-with-git-diff-quiet-td7602438.html

If you don’t want to read all about the bug, just make sure not to use the “—quiet” option with git diff, and you should be all set.


<h2> Issue #4 – The “Polling ignores commits from certain users” option does not work. </h2>

The IAC pipeline needs to be able to monitor the Git repository and run when there is a new integration.  However, because Jenkins itself is integrating version changes back into Git it seems like we would enter an endless loop of builds.

Fortunately, the Git plugin offers an option so that “Polling ignores commits from certain users.”   Using this option, the Jenkins username can be ignored by the polling and we do not enter a loop. 

Unfortunately, this option does not work on it's own.

For details on why this is, see: https://issues.jenkins-ci.org/browse/JENKINS-20607 )

Luckily the work around for this issue is simple.  As described in the above link, all that needs to be done is to enable the Git Plugin’s “force polling using workspace” option.


<h2> Putting it all together </h2>

With the above four issues resolved, this is what the block in Jenkins looks like:

{% codeblock lang:bash %}
# Use Spork to check and then bump version if necessary
knife spork check $COOKBOOK -o $WORKSPACE -p

# Check working directory for a version change
git diff --exit-code
GIT_STATUS=$?

if [ $GIT_STATUS -eq 0 ] 
then
  echo “No changes to push back to Git repo”
else
  echo “Need to merge version change back to Git repo"
  git commit -a -m "From Jenkins IAC"
  git push origin HEAD:master
fi
{% endcodeblock %}
