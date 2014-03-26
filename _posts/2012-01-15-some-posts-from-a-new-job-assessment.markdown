---
author: keymon
comments: true
date: 2012-01-15 04:04:38+00:00
layout: post
slug: some-posts-from-a-new-job-assessment
title: Some posts from a new job Assessment
wordpress_id: 280
---

One year ago I applied for a Linux engineer position in a Internet company abroad.

They did send me a technical assessment with several questions. I took it quite seriously and I did the best I could. Sadly I did not get the job, but I did learn a lot from this.

This was time ago, and now I feel that I can publish some of these replies, they might they are useful for somebody, even as anti-pattern.

One thing that was annoying is that they did not describe the platform enough, and I did have to make a lot of assumptions. But the intention was evaluate me, not implement real solutions.


## Webserver architecture assumptions


To reply the questions, specially OS and PHP version control and Redundant load balancer design, the webserver architecture must be considered. I will design the solutions for the following architecture:



	
  * The servers are located in different physical datacenters or _sites_. I will consider 4 sites.

	
  * The webservers are grouped in _farms_. Farms members should in the same subnet,
connected to an high-performance local network, but on different hardware.
I will consider 16 servers per farm.

	
  * The architecture has a mechanism to share the sessions. For instance we can consider:

	
    * memcached stored sessions.

	
      * One memcached server per webserver.

	
      * One memcached cluster per farm.




	
    * DB backend for the sessions (for sessions it is good a NoSQL DB).

	
      * One DB backend by site. Any webserver recover a session created in the site.
It is possible to configure an inter-site replication.

	
      * It must offer redundancy and HA.







	
  * Other caches implemented can follow the same schema.




### Entradas





	
  * [New job assessment: OS and PHP version control](http://keymon.wordpress.com/2012/01/15/os-and-php-version-contro)

	
  * [New job assessment: Redundant load balancer design](http://keymon.wordpress.com/2012/01/15/new-job-assess…alancer-design/)

	
  * [New job assessment: Routing Script](http://keymon.wordpress.com/2012/01/15/routing-script)

	
  * [New job assessment: MySQL online backup script](http://keymon.wordpress.com/2012/01/15/new-job-assessment-mysql-online-backup-script/)

	
  * [New job assessment: Webcluster logs parsing](http://keymon.wordpress.com/2012/01/15/webcluster-logs-parsing/)


