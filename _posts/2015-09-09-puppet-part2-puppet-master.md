---
layout: post
title: "Puppet part 2: basic puppet master setup"
date: 2015-09-09 00:00:00
categories: puppet howto
---

In [part 1] we went over the absolute basics of setting up a basic manifest file, it’s a start but there’s a lot more useful stuff to learn. Probably the next most important thing to go through is setting up a Puppet Master. Doing so will let you centralize the management of your infrastructure which is useful if you’re managing a large estate.

I’m going to assume you have two fresh CentOS 7 VMs to work with. But maybe you already have a puppet based host you’re looking to connect to a new puppet master. That’s fine but you want to make sure you check the software versions on master and client nodes. In general you want the version on the master to be the same as on the client, or slightly newer on the master. There is no guarantee that a newer client will be able to talk to an older server.

#Server setup

Lets start. On the server install puppet master:

    rpm -ivh http://yum.puppetlabs.com/puppetlabs-release-el-7.noarch.rpm
    yum install puppetserver

There is a certificate exchange that happens between puppet agents and puppet servers as a form of authentication. Maybe you already have your own  public key infrastructure with a certificate authority you’d like to use, however if you’re like me you’ll probably want to let puppet act as a signing authority for puppet related certificates.

Let’s clear out any certs that might have already been created, then get puppet to build a new certificate authority for us:

    vim /etc/puppet/puppet.conf

add the following to the [main] section of the config:

    certname = service.fqdn.or.your.server.example.com

Then:

    systemctl stop puppetmaster
    rm -rf /var/lib/puppet/ssl/*
    systemctl start puppetmaster

Strictly speaking, you don't need to run the stop and rm commands if you've just installed puppetmaster. But maybe you're like me and start searching for howtos once you've already started and got stuck? If so then you need those lines ... 

So anyway, this will start puppet master up again, it’ll see if doesn’t have a cert and regenerate the certificate authority, then read the config to work out what it’s server cert should have for a common name.

A word on hostnames; it’s really important that you have working  forward and reverse DNS for all your puppet nodes nothing will work properly if you don’t. If you don’t have the necessary control over your DNS to setup forward and reverse records, then you’ll need to make sure every host has the relevant hostnames and IP addresses in /etc/hosts as a workaround. Obviously, using /etc/hosts on a large scale is going to get unwieldy so for anything other than a play installation your want to get your DNS sorted.

Next, lets put a simple manifest in place for the clients agent to implement. Edit /etc/puppet/manifests/clienthost.pp

    node ‘clienthost.example.com’ {
      file {‘/tmp/testfile’:
        ensure=>present,
        content=>’foo bar’
      }
    }

We’ll come back to the server in a minute but next, lets go and have a look at the client side

#Client initiation

So, if you don’t already have puppet installed on the client lets do that now:

    rpm -ivh http://yum.puppetlabs.com/puppetlabs-release-el-7.noarch.rpm
    yum install puppet

Now lets configure the agent to talk to the server. Edit /etc/puppet/puppet.conf and add this to [main]:

    server = puppet
    environment = production

Next we run the puppet agent. It’ll connect to the server, but no actual configuration will happen yet:

    sudo puppet agent –onetime

No configuration will have happened because the certificate exchange between the client and server hasn’t happened yet. In our configuration (with no proper PKI system in place) we need to sign the clients certificate from the master before the master will trust it:

#Certificate signing on the server

First lets have a look at the certificates that are waiting to be signed:

    sudo puppet cert list

It’ll list any unsigned certificates the server knows about, hopefully there is a line in there for your client host.

    sudo puppet cert sign client.fqdn.example.org

Re-run the agent on the client

Lets re-run the agent:

    sudo puppet agent –onetime

The agent will spawn into a background process so when this command completes it represents the start of the agent run rather than the end. Monitor /var/log/messages to see how the run is progressing.

In theory that’s it. Check the log to see if the run worked. There should be a new file in tmp based on what we put in the manifest earlier. You can also look at the report the run generated on the server in /var/lib/puppet/reports/ .

Next time we’ll cover setting up the file server part of the puppet master so we can more easily serve files rather than having to specify the content in the manifest.

[part 1]: /puppet/howto/2015/08/06/puppet-part1-getting-started.html
