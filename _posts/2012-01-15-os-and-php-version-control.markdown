---
author: keymon
comments: true
date: 2012-01-15 04:02:10+00:00
layout: post
slug: os-and-php-version-control
title: OS and PHP version control
wordpress_id: 282
categories:
- architecture
- gentoo
- ha
- linux/unix
- Opinion
- sysadmin
- Technical
---

These is one of the proposed solutions for the job assessment [commented in a previous post](http://keymon.wordpress.com/2012/01/15/some-posts-fro…job-assessment).

Note that this was my reply it that moment, nowadays I would change my reply including automatic provisioning and automated configuration  based on puppet, chef... and other techniques. Also something about rollbacks, using LVM snapshots.


### Question




> _Given a 500+ node webcluster in one costumer for one base code. Design a Gentoo/PHP version control. Propose solutions for OS upgrades of servers. Propose a plan and execution for PHP upgrades.  Please explain your choices._




### Solution


About the OS/upgrades I will consider:



	
  * There are a limited number of hardware configurations. I will call them: hw-profile.

	
  * There is a preproduction environment, with servers of each hardware configuration.


In that case:

	
  * 


Each upgrade must be properly tested in the preproduction environment.




	
  * 


The preproduction servers will pre-compile the Gentoo packages
for each hw-profile. Distributed compiling can be set.




	
  * 


There is a local Gentoo mirror and pre-compiled packages repository
in the network, serving the binaries built for each hw-profile.




	
  * 


Each server will have associated its hw-profile repository and install the binaries:




    
    PORTAGE_BINHOST="ftp://gentoo-repository/$hw-profile"
    emerge --usepkg --getbinpkg <package>





The PHP upgrades can be distributed using rsync, in different location for each version,
and activated changing the apache/nginx configuration.

To plan the upgrades (both OS and PHP) I will consider the architecture
explained previously in [Webserver architecture](http://keymon.wordpress.com/2012/01/15/some-posts-fro…job-assessment) section, and the load balancing
solution described in [Redundant load balancer design](http://keymon.wordpress.com/2012/01/15/new-job-assess…alancer-design/).

The upgrade requirements are:



	
  * HA, no lost of service due maintenance or upgrades.

	
  * Each request with an associated session must access to a
webapp version equal or superior than previous request.
This is important to ensure the application consistency
(e.p. an user fills a form that is only available in the last version,
session contains unexpected values...).


The upgrades can be divided in:

	
  * 


_Non-disruptive OS upgrade_: small OS software upgrades that are not related to
the webservice (p.e. man, findutils, tar...). The upgrade can be performed online.




	
  * 


Disruptive OS upgrade: OS software that imply restart the service
or the server (p.e. Apache, kernel, libc, ssl...):




	
    1. It will be upgraded only one member of each farms. First all members number 1,
then number 2...

	
    2. The web service will be stopped during the upgrade.
The other servers in the farm will serve without service disruption.


This method provides homogeneous and little performance impact (100/16 = 6% servers down).

	
  * 


Application upgrade: Clients must access to equal or newer webapp versions:




	
    1. A complete farm must be stopped at the same time.

	
    2. The sessions sticked to this farm will be
served by other farms in the same site (session stickiness to site).
Session data will be recovered from DB backend.

	
    3. The memcached associated to this farm must be flushed.

	
    4. Once upgraded, all servers in the farm are started and can serve to new sessions.


Load balancer stickiness ensure that the new sessions will access only to
the upgraded farm. Except:

	
    * if **all** the servers of the farm fail at the same time after the upgrade.

	
    * if end user manipulates the cookies.


In that case, control code can be added to the application to
invalidate sessions from upper versions. Something like this:

    
    if ($app_version < $_SESSION['app_version'])
     session_destroy()
    elseif ($app_version != $_SESSION['app_version'])
     $_SESSION['app_version'] = $app_session





To perform the upgrades, cluster management tools can be used,
like [MCollective](http://marionette-collective.org/) (good if using puppet),[ func](https://fedorahosted.org/func/), [fabric](http://docs.fabfile.org/0.9.3/)...


