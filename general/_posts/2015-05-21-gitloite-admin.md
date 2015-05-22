---
layout: post
title:  "Some Admin Tasks using gitolite"
date:   2015-05-21 13:18:21
comments: true
---

In this post I will explain how can we do some advanced admin tasks related to server-hosted git repos.
Suppose two scenarios that you may be interested in implementing for your self-hosted git repos.

1. Limiting the number of commits allowed for a developer(s) per a given timespan (day, week, month).
2. Limit the type of content allowed to be pushed to the git repo, For example you may allow senior developers to push whatever they want but they you may be cautios for interns so they deprive them from the right to push binary files :D

This post will be built using one of the greatest git authorization services that I've been able to put my hands on, namely [gitolite](http://gitolite.com/gitolite/index.html)

The tutorial will follow those steps.

1. [Installing gitolite](#install-gitolite)
2. [Create a new Repo and User Groups](#new-repo-group)
3. [Test the new configuration](#test-new-conf)
4. [Implement a Custom VREF](#vref)


<a name="install-gitolite"></a>

### Installing gitolite
In this section we will see how we can install gitolite on a server, I will assume that you will have a server (or a virtual machine using vagrant as I did) with a known IP (I will assume it's 192.168.1.100).
From your main machine we will do the following straightforward steps (I assume you use ubuntu, you can translate to your distro :P)
Before diving in that you need on your maching to create a key-pair for the ssh connection, note here that gitolite get to know who did the commit by the ssh key they use, for your machine you need it to be (git) I've tried to make the admin has a different name but It got me into trouble related to bypassing gitolite, so use ```ssh-keygen``` and create the identity file named ```git```, now take the ```git.pub``` and copy it to the server hosting git.

{% highlight bash %}
sudo apt-get update
sudo apt-get install git
# create a user git
adduser git
su git
# shift to the main directory
cd
git clone git clone git://github.com/sitaramc/gitolite
mkdir -p $HOME/bin
export PATH="$PATH:$HOME/bin"
# Install gitolite
gitolite setup -pk git.pub
# After that step, if you don't get any warnings, everything is fine, otherwise don't neglect them!
{% endhighlight %}

On your machine you need to identify the git server with the new key, so in the ```~/.ssh/config``` append the following

{% highlight todotxt %}
Host gitolite.customgit.dev
     HostName customgit.dev
     User yourusername
     IdentityFile ~/.ssh/git
     IdentitiesOnly yes
{% endhighlight %}

and in the ```/etc/hosts``` you can add the following line, to identify the new host.

{% highlight todotxt %}
192.168.1.100 customgit.dev
{% endhighlight %}

I trust your wisdom in the last two parts to comply to your distro.
Now if everything is fine - which I hope - making the following will supposedly success without any problems

{% highlight todotxt %}
git clone git@gitolite.customgit.dev:gitolite-admin
{% endhighlight %}

If that doesn't happen, I hope you luck with the documentation of gitolite.



<a name="new-repo-group"></a>

### Create a new Repo and User Groups

Once gitolite is setup anything is straightforward, for the first scenario we will create a new repo called ```memcached-python``` with two users groups ```limited``` which indicates the users in that group are limited for a few number of commits, the other is ```unlimited``` and is simply indicating the user is allowed to push any number of commits, the configuration should look like

{% highlight todotxt %}
@limited = scott
@unlimited = superuser
repo memcached-python
    RW+     =   @all
{% endhighlight %}

You can commit and push, if successful you should get something like that

{% highlight todotxt %}
remote: Initialized empty Git repository in /home/git/repositories/memcached-python.git/
{% endhighlight %}

Finally to make the user visible, you have to copy the ```.pub``` file to the ```keydir``` folder of the ```gitolite-admin``` repo and commit then push.


<a name="test-new-conf"></a>

### Test the new configuration

To test the new configuration I've created the a new entry in the ```~/.ssh/config``` to use the new public-private key pair, the entry looked like the following

{% highlight todotxt %}
Host scott.customgit.dev
     HostName customgit.dev
     User Scott
     IdentityFile ~/.ssh/scott
     IdentitiesOnly yes
{% endhighlight %}

To make sure everything is fine you need to make sure that the following command doesn't work (tell you you don't have permission or that the repo doesn't exist)

{% highlight todotxt %}
git clone git@scott.customgit.dev:gitolite-admin
{% endhighlight %}

but this is supposed to work

{% highlight todotxt %}
git clone git@custom.customgit.dev:memcached-python
{% endhighlight %}


<a name="vref"></a>


### VREF 

No it's time for the fun part, gitolite allows for custom constrains using many mechanisms, two important of those are vrefs and hooks.
For the first case (Limiting the number of commits per user in a given time span) I will use VREFS.

let's modify the config to see how can we use VREFS


{% highlight todotxt %}
repo memcached-python
    RW+							=   @all
	-   VREF/CLIMIT/3     		=   @all
{% endhighlight %}

What can you see is that we added the ```-   VREF/CLIMIT/3     		=   @all``` which is the way to tell that: for all users if the virtual reference ```VREF/CLIMIT/3``` is true, damage the push (Indicated by the negative sign)
To write a vref, it's totally straightforward, all you have to do is to write an executable in any scripting language of your choice, the script is supposed to exhaust a given set of arguments and prints the line ```VREF/CLIMIT/3``` or whatever virtual reference to STDOUT, otherwise does nothing, it's crucial to have the script executable and returning zero status code.
If anything is not clear you can consult the docs [here](http://gitolite.com/gitolite/vref.html) and [here for a full list of Env variables used later](http://gitolite.com/gitolite/dev-notes.html) and [here](http://gitolite.com/gitolite/non-core.html).

I will sketch a rough python script that does a certain check to see if the user has committed more than a given number of checks and see if it will grant access or not.

{% highlight python%}
#!/usr/bin/env python
import os

def can_commit(username):
	# implement your logic here and return a boolean

# GL_USER contain the username provided by gitolite
user = os.environ['GL_USER']
if not can_commit(user):
    print "VREF/CLIMIT/3"
exit(0)
{% endhighlight%}

A few notes here

1. ```GL_USER``` contains the username and is passed only using gitolite.
2. You will need to change the process's permission to make sure it's executable and callable.

Now where to put the script.
There are many options

1. If on the server you call ```gitolite query-rc GL_BINDIR``` you will get the root dir of gitolite, you will find a directory VREF, you can through your script there.
2. You can use the ```~/.gitolite.rc``` where there's a commented line that sets the variable ```LOCAL_CODE```, there are two options for the line and both are at the file, whether to set it to any folder or a folder for the admin repo, in all cases don't forget to call ```gitolite setup```.

If you did things well, pushing will work only if the logic in the script allowed you to do so.


The next post will be dedicated to using existing VREFs and custom ones for restricting the size of any commit.
