---
layout: post
title: "Shibboleth SP 2.5.4 & CentOS 7.1"
date: 2015-05-13 00:00:00
categories: Shibboleth Howto
published: true
---

##Prerequisites:

- I’m going to assume you have installed CentOS and that it is connected to the Internet. Also, whilst I do not intend to destroy any data, I also make the assumption that the operating system has been freshly installed.

- It’s going to be helpful to have an Identity Provider around to test with, if you don’t have one already you can --build one by following Shibboleth IDP 3.1.1 on CentOS 7.1-- just use [testshib.org]

##Lets get started

You’re also going to need wget really early on, and also time is really important in shib land so lets have ntp:

    yum install wget ntp
    ntpdate 0.centos.pool.ntp.org
    systemctl start ntpd
    systemctl enable ntpd

As a side note if you have problems with shibboleth authentication after having it running successfully for a long time, check the clocks of the IDP and SP involved, poor time syronization is often the cause of problems.

###Setup shibboleth repository

You can, of course, build the shibboleth software from source, but there is a yum repository hosted on OpenSUSE, so it will be better for us to use that, first we’ll get the key.

    wget http://download.opensuse.org/repositories/security://shibboleth/CentOS_7/repodata/repomd.xml.key -O /etc/pki/rpm-gpg/Shibboleth.key

You should take time to consider how you got the key and how much you trust it, only you can decide the level of trust that is appropriate for your needs. Moving on, let’s setup the repository:

    cat <<EOF > /etc/yum.repos.d/Shibboleth.repo
    [shibboleth]
    name=Shibboleth
    baseurl=http://download.opensuse.org/repositories/security://shibboleth/CentOS_7/

    gpgcheck=1
    gpgkey=file:///etc/pki/rpm-gpg/Shibboleth.key
    EOF
    yum update

###Install the things we’ll need

Yum will take care of all the dependency stuff for us so we don’t need that much:

    yum install httpd mod_ssl shibboleth

At this point you’ll be asked to trust the key we imported earlier, provided you do it’ll install the Shib SP software for us.

##Setup the envrionment

You might already have a web app you want to protect, if so you might want to skip this section because I’m just putting together the necessary firewall and httpd config for us to setup Shibboleth SP.

First let’s allow web traffic through the firewall, open /etc/firewalld/zones/public.xml in your favorite editor and add the following stanzas:

    <service name=”http”/>
    <service name=”https”/>

Reload the firewall:

    systemctl restart firewalld

Setup a demo protected area:

    mkdir /var/www/html/protected
    cat <<EOF> /var/www/html/protected/index.html
    <html><body>Super secret stuff!</body></html>
    EOF

Maybe now is a good time to check httpd kind works, start HTTPD:

    systemctl start httpd
    systemctl enable httpd.service

Then go to http://YourServerHostnameOrIP/protected and check you see the area.

###Configure SELinux

This is really a note for me to improve this section of the document later. For now I’m just going to set SELinux to permissive mode. However, doing so is starting to feel like a modern equivalent of chmod 777, so at some point I need to come back and revisit this. For now, open /etc/sysconfig/selinux in an editor then set SELINUX=permissive

As a side note the official guidance on this at the time of writing is to set it as permissive or disabled: https://wiki.shibboleth.net/confluence/display/SHIB2/NativeSPSELinux

My understanding is that at least with permissive it logs behaviour that didn't meet policy.

###Configure HTTPD

So lets make a location block in our httpd config to wrap shibboleth authentication around our “protected” area, open /etc/httpd/conf/httpd.conf in your editor and add this to the end:

    <Location /protected>
    AuthType shibboleth
    ShibRequestSetting requireSession 1
    require shib-session
    </Location>

We need to start the shibboleth listener daemon:

    systemctl start shibd
    systemctl enable shibd

Now let’s go back to the protected URL. You should see an ConfigurationException error, which proves HTTPD wants shib auth for that URL, and that HTTPD can talk to the Shibboleth daemon. If you see the content you need to revisit your HTTPD config or clear your browser cache, if you see a ListenerException you need to make sure shibd is running and then maybe revisit your SELinux config.

###Configure Shibboleth

Let’s configure the shibd daemon so that it works! Open /etc/shibboleth/shibboleth2.xml in an editor you trust and look for the ApplicationDefaults stanza:

    <ApplicationDefaults entityID=”https://sp.example.org/shibboleth” REMOTE_USER=”eppn persistent-id targeted-id”>

We need to change the entityID here. Shibboleth entityIDs are how the various parts of the shibboleth system refer to each other and so what you have here needs to be globally unique. You’re best off basing the entityID on a domain you control, the URL doesn’t need to serve content however. A safe thing to do is just replace “sp.example.org” with the hostname of your server. Let’s move on to the SSO stanza:

    <SSO entityID=”https://idp.example.org/idp/shibboleth” discoveryProtocol=”SAMLDS” discoveryURL=”https://ds.example.org/DS/WAYF”>
        SAML2 SAML1
    </SSO>

This stanza is really about where your users are coming from. In some installations all your users will come from the same login service (IDP). In others there may be more than one login system that users use to access your service. If you need the multi-idp scenario you’ll need to delete the entityID key value pair and set the discoveryURL to a discovery service. Either one you setup, or one provided by a federation you use (eg: UK Access Management Federation).

To specify your IDP explicitly, change the entityID value to the ID of your IDP, then restart shibd.

    systemctl restart shibd

###Your metadata

It’s possible to hand craft metadata, and I’ve seen people do it, but it’s normally an approach that causes problems. It’s much better to just have the SP software generate the metadata for us:

    wget http://MyHostNameOrIP/Shibboleth.sso/Metadata -O my-sp.metadata.xml

Lets review the metadata, doing so will throw a little light on how the SAML protocol works, plus there is a comment in there we should really remove. Open mp-sp.metadata.xml in your favorite editor. At the start of the file there is a comment about not using the metadata in production without review, when you’re comfortable remove this comment. Moving on, the metadata will start proper with a md:EntityDescriptor stanza. Look at the line, it should have your entityID in it. Later, there is a SPSSODescriptor stanza that will list the protocols we support, then we’ll include the X509 certificate we’ll use to sign our SAML requests, then towards the end we’ll have a bunch of SAML endpoint URLs that are used during communication eg:

    <md:AssertionConsumerService Binding=”urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST” Location=”http://app.serotine.org/Shibboleth.sso/SAML2/POST” index=”1″/>

Great, send the metadata file to your IDP systems administrator, or to your federation, they’ll need to install it at their end. Doing so forms the basis of trust between your service provider and their identity provider. The trust will be based on the X509 certificates in your and their metadata.

###Their metadata

Maybe you’ve joined a federation, if so great they’ll publish metadata for all the other SPs and IDPs in the federation in a way you can consume it. All you’ll need to do is configure your Shibboleth install to import the federation metadata, in /etc/shibboleth/shibboleth2.xml there is a commented out Metadata provider section which is backed by a URI which will probably be useful to you. For my purposes I’m just going to trust some locally maintained metadata for my own IDP, in /etc/shibboleth/shibboleth2.xml:

    <MetadataProvider type=”XML” file=”idp-metadata.xml”/>

Then copy the IDP metadata file to /etc/shibboleth/idp-metadata.xml and restart shibd:

    cp ~/file-I-Got.xml /etc/shibboleth/idp-metadata.xml
    systemctl shibd restart

###Test it

Finally we should be ready to try and login using Shibboleth. Point your browser at http://YouHostNameOrIP/protected and with any luck you’ll see your WAYF or Login page, and by following the process should be back at your protected content.

[testshib.org]: https://testshib.org
