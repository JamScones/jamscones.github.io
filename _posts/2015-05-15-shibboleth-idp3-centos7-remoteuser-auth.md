---
title: "Shibboleth IDP 3.1.1 on CentOS 7.1 with RemoteUser based authentication"
layout: post
date: 2015-05-15 00:00:00
categories: shibboleth howto
---
##Intro: who would want to read this

In part these are notes for myself, but if you want to:

* SAML2 Identity Provider (IDP) for authoritatively asserting claims about a person’s identity
* Hand off the actual authentication to some other mechanism via HTTPD. Eg: htbasic, ldap, CAS, a custom webform, etc
* do this on CentOS 7

Then these instructions should do that for you.

##Prerequisites

* I’m going to assume you have a freshly installed CentOS 7 server with working access to the Internet.
* It’s also going to help a lot if you have a service provider around that you can test the IDP with, if you don’t have a SP already you can build one by following these: Shibboleth SP install on CentOS 7.1. Or just use [testshib.org].

##Lets begin

We’re going to need wget, and time sycronization is important as well. We’ll also need tomcat to host the IDP servlet. Whilst we could configure Tomcat to handle SSL and authenticating users, we’re going to handle that with HTTPD and proxy IDP traffic to Tomcat.

    yum update
    yum install ntp wget httpd mod_ssl lynx
    ntpdate 0.centos.pool.ntp.org
    systemctl start ntpd
    systemctl enable ntpd

The shibboleth constorium state that OpenJDK is supported with Shibboleth IDP, but that Oracle JDK is recommended. They also say if you need help with the product and you’re using OpenJDK, the first thing they are likely to ask is for you to switch to Oracle JDK. By default CentOS uses Open JDK, so we’re going to install Oracle JDK instead. Go to http://www.oracle.com/technetwork/java/javase/downloads/index.html and download the JDK rpm.

    rpm -i jdk-8u45-linux-x64.rpm
    alternatives –install /usr/bin/java java /usr/java/jdk1.8.0_45/bin/java 2
    alternatives –config java
    alternatives –install /usr/lib/jvm java_sdk /usr/java/jdk1.8.0_45/ 2

Set oracle jdk 1.8 JRE as the default runtime.
Let’s finish up by installing tomcat:

    yum install tomcat

###Setup the firewall

Open /etc/firewalld/zones/public.xml and add:

    <service name=”http” />
    <service name=”https” />

###Disable SELInux

I don’t understand SELinux yet, one day I will, but for now I’m going to switch it off and feel dirty about it. Open /etc/sysconfig/selinux in an editor and set SELINUX=permissive .

As a side note, the Shibboleth SP software is not supported with SELinux and I suspect the same statement applies to the IDP software: https://wiki.shibboleth.net/confluence/display/SHIB2/NativeSPSELinux

###Configure HTTPD

We’re going to proxy traffic from HTTPD to Tomcat, so let’s add the proxy rule to the SSL virtual host. Open /etc/httpd/conf.d/ssl.conf and add the following to the virtual host container.

    ProxyPass /idp ajp://localhost:8009/idp retry=5
    <Proxy ajp://localhost:8009>
    Allow from all
    </Proxy>

Lets start HTTPD:

    systemctl enable httpd
    systemctl start httpd

###Install Shibboleth

Shibboleth IDP isn’t in a repository, we need to download the tarball, then run the install script:

    http://shibboleth.net/downloads/identity-provider/latest/shibboleth-identity-provider-3.1.1.tar.gz
    tar zxf shibboleth-identity-provider-3.1.1.tar.gz
    echo “export JAVA_HOME=/usr/lib/jvm/java” > /etc/profile.d/javahome.sh
    sh /etc/profile.d/javahome.sh
    ./shibboleth-identity-provider-3.1.1/bin/install.sh

     

You’ll be asked some questions about your installation, in general accept the defaults, you might want to make sure the hostname and entityID are correct (eg: localhost is wrong).

The compiled webapp doesn’t work with Tomcat by default because it requires the jstl-1.2 library which isn’t supplied by Tomcat, so we download it into our local configuration.

    cd /opt/shibboleth-idp/webapp/WEB-INF/lib
    wget http://download.java.net/maven/1/jstl/jars/jstl-1.2.jar
    /opt/shibboleth-idp/bin/build.sh

###Get the IDP running

We need to pass the installation directory of the IDP to the war at startup, so lets pass it as an option in the tomcat configuration. Open /etc/tomcat/tomcat.conf and add CATALINA_OPTS=”-Didp.home=/opt/shibboleth-idp”

    /opt/shibboleth-idp/bin/build.sh
    chown -R tomcat /opt/shibboleth-idp/logs
    chmod -R a+r /opt/shibboleth-idp/credentials
    chmod -R a+r /opt/shibboleth-idp/conf
    cp /opt/shibboleth-idp/war/idp.war /var/lib/tomcat/webapps/
    systemctl enable tomcat
    systemctl start tomcat

It’ll take about a minute to start tomcat, unpack the IDP and initialize it. You can follow /var/log/tomcat/catalina.datestamp.log to find out when it’s started, once it has you can check the IDP status page to see if the IDP started OK:

    lynx http://127.0.0.1:8080/idp/status

If your IDP installation is working at this point you’ll see some basic information about versions of tomcat/java and some stats for metadata reloads and other IDP related things. Note the IDP won’t be usable yet, this just means it’s a healthy install. If you get an Exception message instead, you need to look into the cause of the exception and fix that. On my first attempt I used OpenJDK and ended up in a series of java Exceptions which I fixed by switching to Oracle JDK.

###Swap metadata

For my purposes I’m just setting up an IDP to test my SP configurations, so I’m just going to manually copy the idp metadata from /opt/shibboleth-idp/metadata/idp-metadata to the SP and copy the SP’s metadata back across to /opt/shibboleth-idp/metadata/ . If you’re joining a federation you’ll probably want to do something different.

I ended up with a file called app.metadata.xml in /opt/shibboleth-idp/metadata . Next, we’ll need to configure the IDP to refer to trust this file. Open /opt/shibboleth-idp/conf/metadata-providers.xml in an editor and add a line like the following in the MetadataProvider section:

    <MetadataProvider id=”App”  xsi:type=”FilesystemMetadataProvider” metadataFile=”/opt/shibboleth-idp/metadata/app.metadata.xml”/>

###Configure authentication

I’m also going to delegate the actual authentication work to HTTPD, so lets configure Shibboleth to use the RemoteUser authenication flow, open /opt/shibboleth-idp/conf/idp.properties and add the following to setup the RemoteUser authentication flow:

    idp.authn.flows= RemoteUser

Now we need to set that up in HTTPD, I’m just going to use htbasic backed by a passwd file for now. Open /etc/httpd/conf.d/ssl.conf and add the following to the virtual host container:

    <Location “/idp/Authn/RemoteUser”>
    AuthType basic
    AuthName Login
    AuthUserFile /etc/httpd/user.file
    require valid-user
    </Location>

Then use htpasswd to create the /etc/httpd/user.file .

Tomcat doesn’t blindly trust the remote_user header passed to it by httpd (why would it) but you can make it do so by adjusting a setting on the AJP connector in /etc/tomcat/server.xml, change the connector on port 8009 to:

    <Connector tomcatAuthentication=”false” port=”8009″ protocol=”AJP/1.3″ redirectPort=”8443″ />

###Restart httpd and tomcat:

    systemctl restart tomcat && systemctl restart httpd

You should now be able to authenticate to your federated service via your IDP installation using the username and password you put on the HTPasswd file.

[testshib.org]: https://testshib.org
