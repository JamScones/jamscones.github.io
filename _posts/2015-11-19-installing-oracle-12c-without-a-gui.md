---
layout: post
title: "Installing Oracle 12c without needing a windowing system on the server"
date: 2015-11-19 00:00:00
categories: Howto
published: true
---

#What?

Oracle 12c is a database server that some people want to use. I need to install it, most of the advice appears to be about how to install it using the pointy clicky interface, I want to keep it all in console.

#Disclaimer

I'm not a Oracle expert. I'm not even a Oracle novice. In the context of Oracle ''I don't know what I'm doing'' And I'm really happy with a future employer knowing that; I'm not interested in work with Oracle Databases. Therefore I cannot guarentee I've done the Oracle part of this correctly. This page is really just the notes I collected whilst trying to get automated install of Oracle Database working and it was hard enough that maybe a line or two of them will be useful to someone else.

#Why do this?

Because who clicks through graphical installers these days. Plus most servers I've worked on don't *have* a windowing system. A lot of them are no where near me and remote X-windows is not a happy place. Also, ultimately I don't want to do the install at all, I want a config management system to do that and that's not going to be able to click at stuff.

#How ... 

## Swap
Before you begin you need to make sure you have the right amount of swap. Often software houses will tell you their software needs this that and the other but the installer will merrily let you install the software and run with less than the recommended amounts. Oracle database isn't like that, the amount of swap you have needs to be correct go to https://docs.oracle.com/database/121/LADBI/pre_install.htm#LADBI7505 and check.

## Hostname

1) set hostname in /etc/hosts (must have both fqdn and shortform)

## SELinux to permissive

*sigh* -- not my idea, I believe this is the advice.

2) set selinux to permissive
12) blank firewall


## Pre-requisite software
3) sudo yum install binutils.x86_64 compat-libcap1.x86_64 compat-libstdc++-33.x86_64 compat-libstdc++-33.i686 \
compat-gcc-44 compat-gcc-44-c++ gcc.x86_64 gcc-c++.x86_64 glibc.i686 glibc.x86_64 glibc-devel.i686 glibc-devel.x86_64 \
ksh.x86_64 libgcc.i686 libgcc.x86_64 libstdc++.i686 libstdc++.x86_64 libstdc++-devel.i686 libstdc++-devel.x86_64 libaio.i686 \
libaio.x86_64 libaio-devel.i686 libaio-devel.x86_64 
 libxcb.i686 libxcb.x86_64 make.x86_64 unixODBC unixODBC-devel sysstat.x86_64

##Kernel parameters
4) kernel parameters ... 
kernel.shmmax = 4294967295
kernel.shmall = 2097152
fs.aio-max-nr = 1048576
fs.file-max = 6815744
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
5) sudo sysctl -p


##Groups and users
6) groupadd -g 54321 oracle
7) groupadd -g 54322 dba
8) groupadd -g 54323 oper
9) sudo useradd -u 54321 -g oracle -G dba,oper oracle
10) sudo usermod -a -G wheel oracle
11) sudo passwd oracle
13) mkdir -p /u01/app/oracle/product/12.1.0/db_1
14) sudo chown -R oracle:oracle /u01
15) sudo chmod -R 775 /u01
16) add following to oracle bash_profile
## Oracle Env Settings 

export TMP=/tmp
export TMPDIR=$TMP

export ORACLE_HOSTNAME=oracle12c.tecmint.local
export ORACLE_UNQNAME=orcl
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/12.1.0/db_1
export ORACLE_SID=orcl

export PATH=/usr/sbin:$PATH
export PATH=$ORACLE_HOME/bin:$PATH

export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib

17) vim /etc/security/limits.conf
oracle	soft	nofile	1024	
oracle	hard	nofile	65536	
oracle	soft	nproc	2047
oracle	hard	nproc	16384
oracle	soft	stack	10240
oracle	hard	stack	32768
18) /etc/security/limits.d/90-nproc.conf

*          -    nproc     16384
root       soft    nproc     unlimited
---

BE ORACLE

ln -s /usr/bin/awk /bin/awk
echo "Red Hat Enterprise Linux AS release 3 (Taroon)" > /etc/redhat-release

18.5) go get the two oracle install files.
root@deboractest:/home/sjones/database# find | grep "\.rsp"
./response/dbca.rsp
./response/db_install.rsp
./response/netca.rsp
root@deboractest:/home/sjones/database# cp response/db_install.rsp ./
root@deboractest:/home/sjones/database# ls
db_install.rsp  install  response  rpm  runInstaller  sshsetup  stage  welcome.html
root@deboractest:/home/sjones/database# 
21) ./runInstaller -silent -responseFile /full/path/to/file/file.rsp (relative paths not supported)

next response file

oracle.assistants.asm|S_ASMPASSWORD=Oracle_01
oracle.assistants.asm|S_ASMMONITORPASSWORD=Oracle_01
oracle.crs|S_BMCPASSWORD=

22) /u01/app/oracle/product/12.1.0/db_1/cfgtoollogs/configToolAllCommands RESPONSE_FILE=<response_file>
23) /u01/app/oraInventory/orainstRoot.sh
24) /u01/app/oracle/product/12.1.0/db_1/root.sh

cp ./stage/ext/lib/libclntsh.so.12.1 /u01/app/oracle/product/12.1.0/db_1/lib/libclntsh.so.12.1
cp ./stage/ext/lib/libclntshcore.so.12.1 /u01/app/oracle/product/12.1.0/db_1/lib/

netca -silent -responseFiler /home/oracle/database/netca.rsp


add the init script

start the server

then

25) something like this to create the db dbca -silent -createDatabase -templateName General_Purpose.dbc -gdbname samdb -sid samdb -responseFile NO_VALUE -characterSet AL32UTF8 -memoryPercentage 30 -emConfiguration LOCAL

