---
author: keymon
comments: true
date: 2010-08-19 12:30:53+00:00
layout: post
slug: linux-nss-libnss-and-nss_ldap-problems-and-posible-solutions
title: Linux NSS (libnss) and nss_ldap problems and possible solutions.
wordpress_id: 168
categories:
- Active Directory
- sysadmin
- Technical
tags:
- activedirectory
- AD
- configuration
- DA
- directory
- dns
- efficiency
- integracion
- ldap
- libnss
- linux
- nscd
- NSS
- nss_ldap
- SFU
- sysadmin
- tip
- tuning
- windows
---

To integrate a Linux system with a centralized user directory (like  Microsoft Active Directory) the usual solution is to configure Kerberos  for Authentication (password/credential checking) and LDAP for  Authorization and Access Control. The "standarized" way to implement  this is using [libpam_krb5](http://linux.die.net/man/8/pam_krb5),  [libnss_ldap](http://linux.die.net/man/5/nss_ldap) (by [padl software](http://www.padl.com/OSS/nss_ldap.html))  and [nscd](http://linux.die.net/man/8/nscd) (from libc).

Kerberos integration works pretty well and I do not have too many issues  with it, but I can not say the same from libnss_ldap and nscd.

In this post I will explain the anoying problems that you can find  using libnss_ldap and nscd, and propose some solutions  and configurations that will make it work properly. I also recomend read a  previous post about the [problems and solutions with connecting an Unix server to  Active directory (Spanish post)](../2010/04/20/integracion-de-sistemas-unix-aix-y-linux-con-directorio-activo-problemas-reales-y-posibles-soluciones/).

Read this article if you are experiencing problems with nscd+libnss_ldap (quoting[ http://www.nico.schottelius.org/blog/nscd-bugs/](http://www.nico.schottelius.org/blog/nscd-bugs/)):



	
  * _Sometimes it consumes 100% cpu (and does not stop that until  being killed)_

	
  * _Sometimes it just crashes._

	
  * _Sometimes it causes users to "vanish" _

	
  * Sometimes it hangs and thus slows down the whole system

	
  * Sometimes it makes all the host work slow

	
  * Sometimes login a host or execute sudo/su takes a lot of time or never logins.

	
  * Sometimes sudo or su dies with "Segmentation Fault"

	
  * Sometimes a simple 'ls' command takes minutes.

	
  * etc...




### <!-- more -->How does all this work?


The NSS (Name Service Switch) is the subsystem (it is formed by modules,  libraries and daemons) that Linux will use to resolve different names:  users, groups, hosts... The base implementation comes with standard C  library (libc), but it can be extendend using _modules_.  [Here you have](http://code.google.com/p/nsscache/wiki/BackgroundOnNameServiceSwitch) a very good description of the library.

Basicly it follows this schema:

[![](http://keymon.files.wordpress.com/2010/08/libnss_default_schema.jpg)](http://keymon.files.wordpress.com/2010/08/libnss_default_schema.jpg)

In the image you can see how libnss works. It is [a shared library](http://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html) that is loaded dynamicaly** in  process space**. [`libnss`](http://www.rage.net/ldap/nss.shtml) will load different modules following the configuration in [`/etc/nsswitch.conf`](http://linux.die.net/man/5/nsswitch.conf). libnss and each  module is a shared library, so its code is shared between processes, but ** internal data and state is stored in process private area**. It will consume the process resources and will be executed into the process threads.

The submodule has in charge execute all the logic needed to resolve the different names. For instance,  with libnss_ldap, it will read the libnss_ldap.conf file, connect to the server, query it and parse the response and return  the values, and it is also responsible to monitor the remote servers, timeout queries. Since each process loads the library, each process does this.

Of cuorse, if you have nscd (name service cache daemon)  running (and the process can access to it), it will query it instead of  calling the submodule. But if the nscd is not available because it  fails, it is not started or something, the process will load all nss  modules...

This behaviour is no really a big deal for simple modules, like local  files. But in the case of libnss_ldap it can be a big problem.  Lets see why...

Firstly, [as  commented here](http://arthurdejong.org/nss-pam-ldapd/design) by _Arthur de Jong_ author of [`nss-pam-ldapd`](http://arthurdejong.org/nss-pam-ldapd/), this original implementation  has several problems (I copy his words):



	
  * _"every executable (that does name lookups) on the system will  load the LDAP libraries and open connections to the LDAP server"_

	
  * _"NSS lookups done in the boot process (e.g. by udev) will  cause the usual timeout mechanism to be invoked (some ugly workarounds  are available)"_

	
  * _"doing hostname lookups through LDAP will cause deadlocks  because the LDAP libraries will need to do hostname lookups to find the  LDAP server (it's a little more subtle than that)"_


On the other hand, the fact of being a shared library has several  disvantages. I quote the [comments in the  source code](http://github.com/keymon/unscd/blob/master/nscd-0.47.c#L27) of [busibox's unscd.c](https://busybox.net/%7Evda/unscd). He talks about nscd, but the  afirmations are true for all processes:



	
  * _"Even if <del>nscd's</del>"..._ program's... "_code is 100.00%  perfect and bug-free, it can still suffer from bugs in libraries it  calls."_...

	
  * _"...it's a multithreaded program which calls NSS libraries.  These libraries are not part of libc, they may be provided by  third-party projects."_

	
  * _"Thus nscd cannot be sure that libraries it calls do not have  memory or file descriptor leaks and other bugs. _ ... _any  resource leak in any NSS library has cumulative effect."_


Also, how the nss API is defined, it is not designed to be used for  networked directory services. As [nsscache's author says](http://code.google.com/p/nsscache/wiki/MotivationBehindNssCache#Problems_with_NSS_and_Directory_Services) (see [also his libnss explanation)](http://code.google.com/p/nsscache/wiki/BackgroundOnNameServiceSwitch)



	
  * _"NSS never fails -- There is no EAGAIN condition in the POSIX  spec for these functions"_

	
  * _"NSS is fast -- it's in the codepath of all these processes,  many interactive _ -- I say more: IT NEEDS TO BE VERY FAST!"


In conclusion I can say about this default design:

	
  * nscd fails more often, so we will lose the cache and force the  programs to load NSS module (like LDAP) libraries and manage the  connections by it self.

	
  * libnss_ldap is quite buggy. Source code is not too  clean and have problems with multithread. Arthur de Jong's author of [nss-pam-ldapd](http://arthurdejong.org/nss-pam-ldapd/) tries to fix this.

	
  * Thus, we are adding third party bugs to our code.

	
  * The process **will be completely locked** while  the name is resolved. All responsability will rely on the nss submodule.

	
  * That means that if there are network delays or a LDAP server is  down we will have big delays. If you have several LDAP servers this can  be a serious problem (delays are multiplied by the number of servers).

	
  * If there is a lock somewhere (nscd, libnss_ldap...), all  applications will freeze ¬_¬.

	
  * If you upgrade the nss_ldap module, or change its  configuration, you have to restart all service to reload the libraries  and configuration.

	
  * Sometimes the libpam library also relies on libnss. This extrapolates the problem to the Authentication process. In my case we had randomly big delays login hosts due this circustance. In fact, it was like the nscd daemon was not being used :?


So, we can say that we are facing a **design problem** and  buggy implementations.


### First  solution: Upgrade to last software versions and tune the software


The very first solution to minimize the problems is to minimize the bugs:



	
  * Have all software in its last version numbers of libc (libnss),  nscd and libnss_ldap

	
  * Tune the configuration. Try to avoid buggy code, minimize locks  and delays, etc. AThe more replicated LDAP servers you have, the lower  times you should use.


I will comment soon in another post our working configuration in our  environment and some tips about it. We are using Suse 11SP2 and Microsoft Active Directory and  right now (19th Agust 2010) a "quite stable" configuration.


### Second  solution: Avoid the problem, replicate the database locally[ ](http://trac.dclinuxapps1.caixagalicia.cg/wiki/SandBox?version=9#Secondsolution:Avoidtheproblemreplicatethedatabaselocally)


You can replicate all the needed users locally: coping it manually,  using an script or using [nsscache](http://code.google.com/p/nsscache/).

nsscache idea is to asyncronously populate a local database  using a python batch tool, and store the result in a local small DB that  is queried by an small simple light nss module.

Actually I really  recommend you read its wiki, it has a very good explanation of the problem, how libnss works and his solution:



	
  * [http://code.google.com/p/nsscache/wiki/BackgroundOnNameServiceSwitch](http://code.google.com/p/nsscache/wiki/BackgroundOnNameServiceSwitch)

	
  * [http://code.google.com/p/nsscache/wiki/ProblemsWithNssAndDirectoryServices](http://code.google.com/p/nsscache/wiki/ProblemsWithNssAndDirectoryServices)

	
  * [http://code.google.com/p/nsscache/wiki/MotivationBehindNssCache](http://code.google.com/p/nsscache/wiki/MotivationBehindNssCache)


Our problem with nsscache is that it is not designed to be used  with Active Directory. It does not support:



	
  * Change the LDAP attribute mapping.

	
  * Nested groups (usual in Active Directory).

	
  * Define only a subset of users/groups to clone.


If I have time, I will try to add those features in the code.


### Third solution:  Use elternative refactorized solutions


The third solution to minimize the problem attacking also its design. You can use the solutions  proposed by:



	
  * Busybox [`unsd`](https://busybox.net/%7Evda/unscd).  It is a _name service cache daemon_, simplier than the original  one, that isolates the nss submodules forking child process, isolating  its bugs aswell. It can completely substitute the normal nscd daemon.

	
  * [nss-pam-ldapd](http://arthurdejong.org/nss-pam-ldapd/): Jong has refactorized the [libnss_ldap module by padl software](http://www.padl.com/OSS/nss_ldap.html), cleaning up all the code. It features:

	
    * An isolated daemon that is resonsible to connect to the LDAP  servers. This daemon tracks which servers are down, testing them  asyncronously (no delays on client processes).

	
    * A small nss module (the smaller it is, the less  bugs) that  connects to the daemon.

	
    * **Drawback**: he removed too much code and nss-pam-ldapd does not  support nested groups :-(





Mixing this two solutions will give you a quite stable solution:

	
  * Simplier and cleaner code implies less bugs.

	
  * Since all name the name resolution logic is isolated in  specific daemons. No more concurrent processes with it's one state.

	
  * Most of potential bugs are isolated.

	
  * Network outages are managed properly and asyncronously.


The global working schema would be as described in this image:

[![](http://keymon.files.wordpress.com/2010/08/libnss_alternate_schema.jpg)](http://keymon.files.wordpress.com/2010/08/libnss_alternate_schema.jpg)


### Fourth solution: Mix all them


I think that the best solution will be clone some important users (services, administrators), use unsd, nss_pam_ldapd and a tuned configuration. I hope implement this option someday and post it here.


### References:





	
  * [http://arthurdejong.org/nss-pam-ldapd/](http://arthurdejong.org/nss-pam-ldapd/)

	
  * [http://code.google.com/p/nsscache/](http://code.google.com/p/nsscache/)

	
  * [http://code.google.com/p/nsscache/wiki/](http://code.google.com/p/nsscache/wiki/BackgroundOnNameServiceSwitch)

	
  * [http://linux.die.net/man/5/nsswitch.conf](http://linux.die.net/man/5/nsswitch.conf)

	
  * [http://linux.die.net/man/8/nscd](http://linux.die.net/man/8/nscd)

	
  * [http://www.padl.com/OSS/nss_ldap.html](http://www.padl.com/OSS/nss_ldap.html)

	
  * [http://www.nico.schottelius.org/blog/nscd-bugs/](http://www.nico.schottelius.org/blog/nscd-bugs/)

	
  * [https://busybox.net/~vda/unscd](https://busybox.net/%7Evda/unscd)

	
  * [https://keymon.wordpress.com/2010/04/20/integracion-de-sistemas-unix-aix-y-linux-con-directorio-activo-problemas-reales-y-posibles-soluciones/](../2010/04/20/integracion-de-sistemas-unix-aix-y-linux-con-directorio-activo-problemas-reales-y-posibles-soluciones/)

	
  * [http://anusf.anu.edu.au/~djh900/nscd.html](http://anusf.anu.edu.au/%7Edjh900/nscd.html)


_… this is another random thinking from keymon ([http://keymon.wordpress.com](../))_
