---
author: keymon
comments: true
date: 2010-06-01 08:58:09+00:00
layout: post
slug: how-does-aix-manage-users-defined-in-different-db-some-gotchas
title: How does AIX manage users defined in different DB? Some Gotchas.
wordpress_id: 96
categories:
- Active Directory
- aix
- sysadmin
- Technical
tags:
- activedirectory
- aix
- celerra
- KRB5files
- KRB5LDAP
- LAM
- NSS
- nss_ldap
- secldapclntd
- ssh
- sudo
- sysadmin
- users
---



We want to known who will work AIX with duplicated users and groups  in  the BUILTIN and LDAP databases.

In Linux, with NSS, the OS  follows the rules defined in /etc/nsswitch.conf,  and **merges** the credentials. If two users entries same  name and different id or  vice versa, it will get the first one. But in  AIX is different.

<!-- more -->We have the following scenario:



	
  * A Active Directory environment with SFU. All groups and users  have its UID/GID.

	
  * An NFS4 server with ACLs and integrated with AD.

	
  * AIX 6.1 TL1 and AIX 6.1 TL4.

	
  * Users and Groups defined in AD.

	
  * Some special and important users (usually service users) [are replicated in local registry. The idea is avoid  dependency with LDAP servers and secldapclntd daemon.](../2010/05/31/aix-using-locally-defined-users-with-kerberos-auth-krb5files/)

	
  * We also have an NFS4 server that is integrated with active  directory and implements ACLs (EMC Celerra)


On aix to create local users, I used this command mkuser -R  KRB5files registry=KRB5files SYSTEM=KRB5files <user> so it  uses the local registry but authenticates against Kerberos.


> **Note:** always use KRB5files as registry. If you use  compat, the local password policies will be applied, [and this might be a  problem](https://keymon.wordpress.com/2010/05/31/aix-using-locally-defined-users-with-kerberos-auth-krb5files/).


When I create the user, I do not add some of the LDAP defined groups to  the user, so querying one security method (LDAP) or the other (compat)  will return different groups. You can check this with lsuser:

    
    # mkuser -R KRB5files registry=KRB5files SYSTEM=KRB5files groups=staff,unixadm jhon
    
    # lsuser -R LDAP -a groups registry SYSTEM jhon
    jhon groups=unixuser,unixadm registry=LDAP SYSTEM=KRB5files
    # lsuser -R KRB5files -a groups registry SYSTEM jhon
    jhon groups=staff,unixadm registry=KRB5files SYSTEM=KRB5files
    # lsuser -a groups registry SYSTEM jhon
    jhon groups=staff,unixadm registry=KRB5files SYSTEM=KRB5files
    


As you can see, in LDAP the user belongs to unixuser and unixadm,  but locally to staff and unixadm.

In the last command you see that, since as the user jhon has  stanza registry=KRB5files defined in /etc/security/user,  KRB5files will be the default DB queried to get its  credentials.


### Duplicated user in  BUILTIN and LDAP databases[
](http://trac.dclinuxapps1.caixagalicia.cg/wiki/SandBox?version=5#DuplicateduserinBUILTINandLDAPdatabases)


Does AIX merges the credentials like NSS on Linux does? If the answer is  "yes", the groups should be unixuser, unixadm and staff.  If it is "no", the locally defined users should only have the  credentials for database defined in its registry attribute.  Let's see what id says:

    
    # chuser -R compat registry=KRB5files jhon
    # id jhon
    uid=204(jhon) gid=1(staff) groups=10103(unixadm)
    # chuser -R compat registry=LDAP jhon
    # id jhon
    uid=204(jhon) gid=1(staff) groups=10103(unixadm)
    


**WTF????** id command always uses the compat!!!

Let's test if these credentials are real, checking whether the file  permissions are applied, etc:

    
    # chuser -R compat registry=KRB5files jhon
    # lsuser -a groups registry SYSTEM jhon
    jhon groups=staff,unixadm registry=KRB5files SYSTEM=KRB5files
    # id jhon
    uid=204(jhon) gid=1(staff) groups=10103(unixadm)
    # su jhon -c id
    uid=204(jhon) gid=1(staff) groups=10103(unixadm)
    
    # echo "this is a test" > /tmp/testfile
    # chmod 770 /tmp/testfile
    # chgrp unixuser /tmp/testfile
    # ls -l /tmp/testfile
    -rwxrwx---    1 root     unixuser         15 May 31 12:29 /tmp/testfile*
    
    # su jhon -c cat /tmp/testfile
    cat: 0652-050 Cannot open /tmp/testfile.
    
    # chgrp unixadm /tmp/testfile
    # su jhon -c cat /tmp/testfile
    this is a test
    


So, with registry=KRB5files, it uses BUILTIN DB.

    
    # chuser -R compat registry=LDAP jhon
    # lsuser -a groups registry SYSTEM jhon
    jhon groups=unixuser,unixadm registry=LDAP SYSTEM=KRB5files
    # id jhon
    uid=204(jhon) gid=1(staff) groups=10103(unixadm)
    # su jhon -c id
    uid=204(jhon) gid=1(staff) groups=10103(unixadm)
    
    # chgrp unixuser /tmp/testfile
    # su jhon -c cat /tmp/testfile
    cat: 0652-050 Cannot open /tmp/testfile.
    # chgrp unixadm /tmp/testfile
    # su jhon -c cat /tmp/testfile
    this is a test
    


And with registry=LDAP, it also uses BUILTIN DB. So, lsuser returns  incorrect groups.

Now we can test these credentials with "sudo":

    
    # lsuser -a groups registry SYSTEM jhon
    jhon groups=unixuser,unixadm registry=LDAP SYSTEM=KRB5files
    # echo "%unixuser       ALL=(ALL)       NOPASSWD:/usr/bin/id" >> /etc/sudoers
    # su jhon -c "sudo /usr/bin/id"
    Password:
    Sorry, try again.
    # cp /etc/sudoers /etc/sudoers.bak ; sed 's/unixuser/unixadm/' </etc/sudoers.bak > /etc/sudoers
    # su jhon -c "sudo /usr/bin/id"
    uid=0(root) gid=0(system) groups=2(bin),3(sys),7(security),8(cron),10(audit),11(lp)
    


We can see the same behaviour: It will only use the groups from the  BUILTIN db (the ones that id return).

**First conclusion**: in id command, basic file  permissions, sudo and other client applications, the registry attribute is not taken in account. It will always use the local user  credentials.   Well this was an _undocumented_ but is Ok. AIX is not prepared to  support the same user defined in the DB of different methods, it will  use the BUILTIN DB by default.


### Can you mix BUILTIN and  LDAP users and groups?[
](http://trac.dclinuxapps1.caixagalicia.cg/wiki/SandBox?version=5#CanyoumixBUILTINandLDAPusersandgroups)


Then one question raises: Can I add LDAP groups to local users? In linux  with NSS you can, but in AIX the reply is "NO":

    
    # lsuser -a groups registry SYSTEM jhon
    jhon groups=staff,unixadm registry=KRB5files SYSTEM=KRB5files
    # chuser groups=staff,unixadm,unixuser jhon
    3004-686 Group "unixuser" does not exist.
    


I think that the restriction here are the mkuser and mkgroup commands. If you edit the files manually, maybe it works, but I will  not accept such _hack_.

Anyway, you can create the group locally with same id and assign it to  the local user:

    
    # lsgroup -a id "unixuser"
    unixuser id=10102
    # mkgroup id=10102 unixuser
    # chuser groups=staff,unixadm,unixuser jhon
    # lsuser -a groups registry SYSTEM jhon
    jhon groups=staff,unixadm,unixuser registry=KRB5files SYSTEM=KRB5files
    # lsgroup unixuser
    unixuser id=10102 admin=false users=jhon registry=files
    


Note that other LDAP users still belong to unixuser and can  access to files that belong that group:

    
    # lsuser -a groups registry SYSTEM testaix
    testaix groups=unixuser registry=KRB5LDAP SYSTEM=KRB5LDAP OR compat
    # ls -l /tmp/testfile
    -rwxrwx---    1 root     unixuser         15 May 31 12:29 /tmp/testfile*
    # su testaix -c "cat /tmp/testfile"
    this is a test
    


But, does applications like sudo or sshd work? well it depends. I found  that with sudo-1.6.9p15-2noldap (from AIX ToolboxForLinuxApplications? 05.2009) works:

    
    # rpm -q sudo
    sudo-1.6.9p15-2noldap
    # echo "%unixuser       ALL=(ALL)       NOPASSWD:/usr/bin/id" >> /etc/sudoers
    # su testaix -c "sudo /usr/bin/id"
    uid=0(root) gid=0(system) groups=2(bin),3(sys),7(security),8(cron),10(audit),11(lp)
    


But previous versions (like sudo-1.6.7p5-3) not:

    
    # rpm -q sudo
    sudo-1.6.7p5-3
    # echo "%unixuser       ALL=(ALL)       NOPASSWD:/usr/bin/id" >> /etc/sudoers
    # su testaix -c "sudo /usr/bin/id"
    uid=0(root) gid=0(system) groups=2(bin),3(sys),7(security),8(cron),10(audit),11(lp)
    


In general, I tested:



	
  * sudo 1.6.7p5-3: fails

	
  * sudo 1.6.9p15-2: Ok

	
  * openssh  4.3.0.5201: ok

	
  * openssh  5.2.0.5300: ok


I do not known why sudo 1.6.7p5-3 was failing.

So, **Second conclusion**: users and groups from different  sources can not be mixed, but you can create the groups locally. File  permissions work properly for groups. Applications that implement  authorization and authentication must be tested.

Probably, an application that needs to known the users from a group will  fail.


### The NFS4 ACLs problem


Well, we known that with duplicated users the BUILTIN has preference,  with duplicated groups it works more or less Ok.  But, on the other  hand, I found a really weird behaviour when we were accessing the  mounted NFS4 directory.

We have a NFS4 directory mounted with these options:

    
    nfsserver /an_nfs4_directory /an_nfs4_directory nfs4   May 05 10:39 soft,intr,acl,retry=10,retrans=5,timeo=5,vers=4,rw
    


and this directory uses ACLs, where group unixuser was able to  read but not unixadm (the output has been modified to remove real  names):

    
    # aclget /an_nfs4_directory
    *
    * ACL_type   NFS4
    *
    *
    * Owner: admin
    * Group: Domain Admins
    *
    g:nobody(Domain Admins@caixagalicia.cg):        a       rwpRWxDaAdcCos  fidi
    g:nobody(unixuser@caixagalicia.cg):        a       rRxacs  fidi
    g:nobody(writers@caixagalicia.cg):        a       rwpRWxaAdcs     fidi
    


With these permissions, the user shouldn't be able to read the  directory, but:

    
    $ whoami
    jhon
    $ cd /an_nfs4_directory
    $ pwd /an_nfs4_directory
    /an_nfs4_directory
    $ ls -la
    d------rwx    8 admin   Domain A       2048 May 31 13:26 ./
    dr-xr-xrwx    9 admin   Domain A       1024 May 03 09:29 ../
    -------rwx    1 admin   Domain A       8310 May 17 19:34 somefile
    $ cat somefile
    This file is on NFS4
    


WTF???? It is able to access to the NFS directory!!!

I will try to simulate this locally with JFS2 ACLs. But note that local  JFS2 ACLS are not the same than NFS4 ACLS:

    
    # aclput /tmp/testfile <<EOF
    *
    * ACL_type   AIXC
    *
    attributes:
    base permissions
        owner(root):  rwx
        group(sys):  rwx
        others:  ---
    extended permissions
        enabled
        permit   rw-     g:unixuser
    EOF
    # su jhon -c cat /tmp/testfile
    cat: 0652-050 Cannot open /tmp/testfile.
    


But it doesn't work... So this must be something related to the NFS4 and  its ACLs.

One could say: _"The problem is that the credentials to access the  NFS4 directory are checked in the nfs server side"_... but that is  false. First, to implement that, you should use Kerberos. Secondly, I  tested that: If I try create the unixuser group locally and I  add to it a local user which **does not exists in the Active  Directory**, the local user is able to access to the NFS  directory. For instance root:

    
    # cat /an_nfs4_directory/somefile
    cat: 0652-050 Cannot open  /an_nfs4_directory/somefile.
    # mkgroup id=10102 users=root unixuser
    # cat /an_nfs4_directory/somefile
    This file is on NFS4
    


And jhon is still able to access:

    
    # su jhon -c cat /an_nfs4_directory/somefile
    This file is on NFS4
    


Does somebody have an idea about why is happening this? I am completely  lost.

Meanwhile, **Third conclusion**: NFS4 ACLs on AIX will be  checked against all the configured DBs.

Actually, for me, this behaviour is really convenient. I can clon  service and administrators users locally to avoid impact of active  directory outages, but I do not need to clone all the groups.


### References[
](http://trac.dclinuxapps1.caixagalicia.cg/wiki/SandBox?version=5#References)





	
  * [http://publib.boulder.ibm.com/infocenter/aix/v6r1/index.jsp?topic=/com.ibm.aix.files/doc/aixfiles/user.htm](http://publib.boulder.ibm.com/infocenter/aix/v6r1/index.jsp?topic=/com.ibm.aix.files/doc/aixfiles/user.htm)

	
  * [http://www.ibm.com/developerworks/aix/library/au-compaixsolaris/index.html](http://www.ibm.com/developerworks/aix/library/au-compaixsolaris/index.html)

	
  * [http://www.ibm.com/developerworks/aix/library/au-filesys_NFSv4ACL/index.html](http://www.ibm.com/developerworks/aix/library/au-filesys_NFSv4ACL/index.html)



