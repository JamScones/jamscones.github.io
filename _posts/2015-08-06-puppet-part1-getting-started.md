---
title: "Puppet part 1: Getting started with a basic manifest file"
date: 2015-08-06 00:00:00
categories: puppet howto
layout: post
---

#Introduction

First, the PuppetLabs documentation is great! It’s very complete and I’ve nearly always found the answer to my Puppet questions in it, but when I was first starting with Puppet I found it quite difficult to sort through. There are a bunch of different concepts going on in the Puppet world, and the documentation tends to reference itself quite a lot. I’d often find myself with 10 or so tabs open somewhere on the puppet labs site and feeling a little lost about how to achieve things that I thought should be quite simple.

So, if you’re that person: already sold on automated system deployment/configuration, already sold on puppet, having trouble getting started, then you’re the person I’m aiming this series at.

#This series

I hope to cover a bunch of the puppet concepts I found useful. I’m going to start with a fairly simple site.pp file that manages the local server. It’s about as basic as you can get with Puppet, but even this is still an incredibly useful tool. You get:

* the ability to rebuild your server in the event of disaster (provided you have a copy of your site.pp)
* the possibility of being able to revert to earlier system configurations in a managed way, by keeping the old versions of the site.pp
* a more manageable way of having multiple sysadmins on a single server.

Later in the series I will cover:

* [Part 2] - puppet master setup
* puppet file server
* puppet environments
* using puppet modules
* writing puppet modules.

#Install puppet agent

These instructions are based on CentOS 7, and need to be run as root.

First install the puppet agent:

    rpm -ivh http://yum.puppetlabs.com/puppetlabs-release-el-7.noarch.rpm
    yum install puppet

You’ll need to agree to an install and trust a couple of keys so you can use the PuppetLabs repository.

#Our first manifest file

Next we’ll put together a very simple manifest file. In your favorite text editor open a new file called manifest.pp and enter the text below:

    package {“httpd”:
      ensure=>installed,
      before=>Service['httpd'],
    }
    service {“httpd”:
      ensure=>running,
    }
    file {“/var/www/html/index.html”:
      ensure=>present,
      content=>”Foo bar”,
    }

Save the file, close the editor. Puppet manifest files focus on declaring the state the system should be in rather than the defining the processes that need to happen to get to those states. So in the file above we’re saying that HTTPD should be installed, should be running, and index.html should contain “Foo bar”.

#Run puppet

Let’s get Puppet to apply the file, run:

    puppet apply manifest.pp –onetime

You should get this:

    [root@localhost ~]# puppet apply manifest.pp –onetime
    Notice: Compiled catalog for localhost.localdomain in environment production in 0.67 seconds
    Notice: /Stage[main]/Main/Package[httpd]/ensure: created
    Notice: /Stage[main]/Main/Service[httpd]/ensure: ensure changed ‘stopped’ to ‘running’
    Notice: /Stage[main]/Main/File[/var/www/html/index.html]/ensure: created
    Notice: Finished catalog run in 5.10 seconds
    [root@localhost ~]#

It worked! Looking at the output our manifest file compiled OK, Puppet installed httpd, started it, and created the index.html. If you point a web browser at your host you should see the “Foo bar” message we put in our not-so-valid index.html file.

Look back at the manifest file. It’s mostly self explanatory: make sure httpd is installed, make sure it’s running, make sure index.html has the content “Foo bar”. The only other thing to note is the before=>Service['httpd'] line, this ensures that the stanza that installs HTTPD is satisfied before the stanza that starts httpd.

#What else …

Say we were also setting up a custom HTTPD config in our puppet file. We’d have a stanza like:

    file {“/etc/httpd/conf/httpd.conf”:
      ensure=>present,
      notify=>Service['httpd'],
      content=>” …… ”
    }

Maybe when this runs HTTPD is already started so the service stanza that is also in the file has no effect. Therefore we need a way to tell Puppet to restart HTTPD when the config file changes. This is what the “notify=>Service['httpd']” line does. If the content of the file changes, Puppet knows to notify the HTTPD service which has the effect of restarting it.

#Finally

So that’s a basic manifest file that sets up a very simple web server. It is however a long way from ideal. Probably the main thing that’s inconvenient about this is that we need to specify the content of each file we modify in the manifest file which will obviously get cumbersome. But don’t worry! We’ll fix that later in the series.

[Part 2]: /2015/09/09/puppet-part2-puppet-master.html
