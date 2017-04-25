---
title: "Puppet part 3: file server"
date: 2015-11-15 08:33:00
layout: post
categories: puppet howto
published: false
---

This is really easy, but it took me a long time to find out about. When I was trying to find out how to distribute files using puppet I'd normally hit upon people who were recommending creating a module and then serving the items out of there which seemed a bit broken to me. Especially when people would use example module names that implied they had created the module purely for the purpose of serving files.

Turns out those people are right. I mean not the bit about having a place holder module for your files, but the idea that most if not all of the files you're going to distriubte should be in a module is correct. I didn't get that until later; I just wanted to not have to put the content of my files in the main manifest because I had that in version control and there are certain things I don't like putting in version control (credentials, binaries, etc). So for the person who is essentially in the place that I was; hurriedly running up configs trying to get something to work:

# You can just do this ...

There is such a thing as the puppet fileserver, currently it's the only supported transfer source for the "source" attribute of the "file" type.

Create a file called /etc/puppet/fileserver.conf, or edit it if already there (if it's already there you probably don't need this post the format is that simple).

Then add a stanza line the following:

    [share_name]
      path /path/to/share
      allow *

Then from the manifest you'll be able to include that file with a stanza like:

    file {"/etc/confyconfig.conf":
      ensure=>present,
      source=>"puppet:///share_name/filename"
    }

# You probably shouldn't be doing that

As I said above, this was the first place I went to and later I realised that it was conceptually wrong. But as it happens the next part of this series is about puppet modules so this will serve as a nice lead in.

Puppet modules are essentially about abstracting your infrastructure into a series of conceptual objects. If you're from an object orientated programming background you should get this concept pretty quickly, if not I'd say if you're on the technical end of IT you should *get* a concept of how object orientated programming works. Also, if you're from a object orientated programming background I would advise you to be a little soft when applying your understanding of OO principles to puppet classes/modules. So far I've found the definition of "module" & "class" are used more or less interchangable in puppet land. 

Say you're running up a web application cluster. You've got a bunch of frontend web servers, a load balancer, a bunch of database servers, and some other things knocking around like admin portals. You're going to want lots of these things to be as identical as possible so the administration is easier at a small scale and sane at a larger scale. So you're going to end up with a class  that repersents the frontend box, then apply that class to each instance. The logic follows that any file that a frontend class needs will be in the module, and that in order to keep the frontends consistent everything that should make up a frontend box should be in the class. Therefore in theory there aren't that many good reasons to want to do the file server scenario we configure above.

# Exceptions

Passwords, keys, binary files that don't sit well in version control. But then are these really exceptions from a puppet point of view? There's nothing that says you have to layer your config management infrastructure on a source code management system, even if it's an extremely good idea and common practice.

If we were keeping these things out of puppet because we were worried about the puppet server being used as an attack vector (which is certainly worth considering). We should also think about this: the puppet server effectively has root on all your boxes anyway, so if I have compromised the puppet master what's to stop me writing a manifest that installs a program that runs as root and reveals the sensitive stuff that you avoided putting on the puppet master puppet? Unless you're not running puppet as root (which I think is unusual) if someone has compromised your puppet infrastructure you've already lost so you might as well take the gain of being able to use it to manage sensitive information, and put effort into not allowing the puppet master to be compromised.
