+++ 
draft = false
date = 2015-08-04
title = "Memcached logging (and others) under Systemd on RHEL7"
description = ""
summary = "How to cleanly gather logs for memcached."
slug = ""
authors = []
tags = ["memcached","logging","systemd"]
categories = []
externalLink = ""
series = []
+++

> This post was originally published at my old blog Distracted IT on Blogger.com

I've been getting into RHEL7 lately (yay, a valid reason to get into at work!) and this means learning about systemd. This post is not about systemd... at least its not another systemd tutorial. This post is about how I got memcached to emit its logging to syslog while running under systemd, and how to configure memcache to sanely log to a file that I can expose via a file share.  
  
Its 2:21am, so this is gonna be quick. I wrote this because I needed memcached, and in response to some [bad advice on Stack Overflow](http://serverfault.com/questions/208538/how-to-specify-the-log-file-for-memcached-on-rhel-centos/710452#710452).  

The usual configuration... it hasn't changed much **BUT** do be aware that OPTIONS doesn't appear to allow a shell script (so no shell redirections) and that I've specified -vv in order to get some logs.  

```bashsession
# cat /etc/sysconfig/memcached  
PORT="11211"  
USER="memcached"  
MAXCONN="1024"  
CACHESIZE="64"  
OPTIONS="-l 127.0.0.1 -vv"  
```

First off, note that we can actually get information using `journalctl`; this should capture anything sent to a systemd service's stdout, by default. Although after this body of work, the logs will be sent to syslog, instead of to the journal, so this will be the last we'll see of them in the journal. 

```bashsession
# journalctl --since '2012-01-01' _SYSTEMD_UNIT=memcached.service  
-- Logs begin at Fri 2015-07-10 11:00:21 NZST, end at Tue 2015-08-04 02:23:05 NZST. --  
Aug 03 23:36:49 HOSTNAME memcached[4318]: slab class   1: chunk size        96 perslab   10922  
Aug 03 23:36:49 HOSTNAME memcached[4318]: slab class   2: chunk size       120 perslab    8738
...
```

Here is the default / base service definition for memcached. We don't touch it. Note that it still includes /etc/sysconfig/memcached  

```bashsession
# cat /usr/lib/systemd/system/memcached.service  
[Unit]  
Description=Memcached  
Before=httpd.service  
After=network.target  
  
[Service]  
Type=simple  
EnvironmentFile=-/etc/sysconfig/memcached  
ExecStart=/usr/bin/memcached -u $USER -p $PORT -m $CACHESIZE -c $MAXCONN $OPTIONS  
  
[Install]  
WantedBy=multi-user.target  
```

Our changes can be dropped into a file under /etc/systemd/system/...  

```bashsession
# cat /etc/systemd/system/memcached.service.d/local.conf  
[Service]  
StandardOutput=syslog  
StandardError=syslog  
SyslogIdentifier=memcached  
SyslogFacility=local1  
SyslogLevel=debug  
SyslogLevelPrefix=false  
```

Reload the systemd configuration and restart the service that it is about.  

```bashsession
# systemctl daemon-reload  
# systemctl restart memcached.service  
```
  
Restart and test that its running and its picked up our file.  

```bashsession
# systemctl status memcached  
memcached.service - Memcached  
   Loaded: loaded (/usr/lib/systemd/system/memcached.service; enabled)  
  Drop-In: /etc/systemd/system/memcached.service.d  
           └─local.conf        <---------- NOTE  
   Active: active (running) since Tue 2015-08-04 01:07:50 NZST; 7s ago  
 Main PID: 3842 (memcached)  
   CGroup: /system.slice/memcached.service  
           └─3842 /usr/bin/memcached -u memcached -p 11211 -m 64 -c 1024 -l 127.0.0.1 -vv  
  
... _a tail of log messages show here_  
```  

> A better way to view the configuration for a service is to instead use `systemctl cat memcached.service`; you'll see this will include the configuration from under /usr/lib/... as well as our local.conf.

Rsyslogd uses its imjournal module to read logs from journald; make sure you have this configured if you've brought your rsyslog config from a previous version of RHEL. I fell into this trap and I noticed I could see logs from 'logger', but not from journald. My method of prompting memcache to emit some logs was simply to restart it (with -vv you get plenty)  

```bashsession
# echo "local1.debug  /var/log/memcached/memcached.log" >> /etc/rsyslog.d/memcached.conf  
# mkdir /var/log/memcached  
# systemctl restart rsyslog.service  
# systemctl status rsyslog.service  
```
  
Don't forget log rotation  
  
```logrotate.conf
# cat /etc/logrotate.d/memcached  
/var/log/memcached/memcached.log {  
    daily  
    rotate 3  
    dateext  
    missingok  
    create 0640 root root  
    compress  
    delaycompress  
    postrotate  
        /bin/kill -HUP `cat /var/run/syslogd.pid 2> /dev/null` 2> /dev/null || true  
    endscript  
}
```
