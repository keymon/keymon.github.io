---
author: keymon
comments: true
date: 2010-05-31 08:26:34+00:00
layout: post
slug: aix-using-locally-defined-users-with-kerberos-auth-krb5files
title: 'AIX: Using locally defined users with Kerberos auth (KRB5files). '
wordpress_id: 88
categories:
- Active Directory
- aix
- fast-tip
- sysadmin
tags:
- 3004-333
- ADMCHG
- aix
- integracion
- KRB5files
- sysadmin
- user
---



You want to have a user defined locally but delegate the authentication  to a Kerberos server (like active directory). That is ok, specially  since secldapclntd [is not the most reliable component on aix.](../2010/04/23/monitoring-and-restarting-secldapclntd-daemon-in-aix/)

But be careful, if you define a user in the compat registry instead of  KRB5files (but with SYSTEM=KRB5files), like in this command:

    
    mkuser -R KRB5files SYSTEM=KRB5files <user>
    


you will find that the local password policies will be applied to the  user. This is a incorrect behaviour, because AIX does not manage the  password.

For instance, despite having SYSTEM=KRB5files, the new user  will have the ADMCHG attribute defined in its stanza in /etc/security/passwd

    
    jhon:
            password = *
            lastupdate = 1275046476
            flags = ADMCHG
    


From man pwdadm:

    
    ADMCHG
       Resets the ADMCHG attribute without changing the user's password. This forces the user to change passwords
       the next time a login command or an su command is given for the user. The attribute is cleared when the
       user specified by the User parameter resets the password.
    


With this attribute set and SYSTEM=KRB5files, we will get this  error if we try to login (for instance, via SSH):

    
    May 31 10:10:38 aixhost01 auth|security:info sshd[585730]: Password can't be changed for user jhon: [compat]: 3004-333 A password change is required. 3004-320 Only the system administrator can change
    May 31 10:10:38 aixhost01 auth|security:info sshd[585730]: Failed password for jhon from 1.2.3.4 port 62018 ssh2
    May 31 10:10:38 aixhost01 auth|security:info syslog: ssh: failed login attempt for jhon from acomputer.localdomain
    


To avoid this, you can reset the password, or execute pwdadm -c jhon,  but the best solution is simply change the registry:

    
    chuser registry=KRB5files jhon
    


Or:

    
    USER=jhon
    chuser expires=0 maxage=0 maxexpired=-1 minage=0 loginretries=-1 registry=KRB5files $USER
    pwdadm -c $USER
    
    







Other tipical errors of this:

Sep 15 09:56:56 tcmurexappl1 auth|security:info sshd[344552]: Password can't be changed for user jhon: [compat]: 3004-330 Yourencrypted password is invalid. \r3004-320 Only the system administrator can c
Sep 15 09:56:14 tcmurexappl1 auth|security:info sshd[213376]: Failed password for jhon from 168.1.1.x port 1632 ssh2
Sep 15 09:56:14 tcmurexappl1 auth|security:info syslog: ssh: failed login attempt for jhon from apc.mycompay.org
Sep 15 09:56:18 tcmurexappl1 auth|security:debug sshd[213376]: [krb_authenticate] Error in getting TGT ...
Sep 15 09:56:18 tcmurexappl1 auth|security:debug sshd[213376]: Preauthentication failed

To solve it:

USER=jhon
chuser expires=0 maxage=0 maxexpired=-1 minage=0 loginretries=-1 registry=KRB5files $USER
pwdadm -c $USER


