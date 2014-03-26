---
author: keymon
comments: true
date: 2010-04-23 11:03:48+00:00
layout: post
slug: monitoring-and-restarting-secldapclntd-daemon-in-aix
title: Monitoring and restarting secldapclntd daemon in AIX
wordpress_id: 67
categories:
- aix
- fast-tip
- monitoring
- script
- sysadmin
tags:
- 3001-803
- acti
- AD
- aix
- bug
- cron
- DA
- failure
- integracion
- ldap
- monitor
- monitoring
- resolver
- restart-secldapclntd
- scripting
- secldapclntd
- shell
- sysadmin
- tip
- trick
---

The AIX service [secldapclntd ](http://www.regatta.cs.msu.su/doc/usr/share/man/info/ru_RU/a_doc_lib/cmds/aixcmds5/secldapclntd.htm)_"Provides and manages connection between the AIX LDAP load module of the local host and LDAP Security Information Server, and handles  transactions from the LDAP load module to the LDAP Security Information Server."_

This services fails too often. Each new version of AIX, brings new failures in this services. Failures appear more often if the LDAP server has a lot of users and groups.

When it fails:



	
  * sometimes does not reply, its hung

	
  * sometimes it consumes all CPU and lasts a lot to reply

	
  * sometimes it simply dies with a core.


This script will check and monitor it and restart it if necesary.

You can test this script stoping the service:

    
    kill -STOP $(ps -fea| grep -v grep |grep /usr/sbin/secldapclntd| awk '{print $2}' )
    


You can add an entry to cron to execute it:

    
    if crontab -l | grep /usr/local/bin/check-secldapclntd.sh; then
       echo "Already configured."
    else
      crontab -l > /tmp/$$.crontab
      cat >> /tmp/$$.crontab <<EOF
    # Check secldapclntd each 5 minutes
    5,10,15,20,25,30,35,40,45,50,55 * * * * /usr/local/bin/check-secldapclntd.sh check-and-restart > /dev/null
    EOF
      crontab /tmp/$$.crontab
      rm /tmp/$$.crontab
    fi
    


And here goes the script (/usr/local/bin/check-secldapclntd.sh):



	
  * [Source: http://github.com/keymon/snippets/blob/master/scripts/aix/check-secldapclntd.sh](http://github.com/keymon/snippets/blob/master/scripts/aix/check-secldapclntd.sh)

	
  * Download (Right Click > save as...): [http://github.com/keymon/snippets/raw/master/scripts/aix/check-secldapclntd.sh](http://github.com/keymon/snippets/raw/master/scripts/aix/check-secldapclntd.sh)


