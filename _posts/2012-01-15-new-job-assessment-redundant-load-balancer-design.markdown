---
author: keymon
comments: true
date: 2012-01-15 04:05:25+00:00
layout: post
slug: new-job-assessment-redundant-load-balancer-design
title: 'New job assessment: Redundant load balancer design'
wordpress_id: 285
categories:
- architecture
- ha
- linux/unix
- Misc
- sysadmin
- Technical
- web
tags:
- architecture section
- load balancer
- open source solution
---

These is one of the proposed solutions for the job assessment [commented in a previous post](http://keymon.wordpress.com/2012/01/15/some-posts-fro…job-assessment).


> _Using an Open Source solution, design a load balancer configuration that meets: redundancy, multiples subnets, and handle 500-1000Mbit of syn/ack/fin packets. Explain scalability of your design/configs._


The main problem that the load balancer design must solve in
web applications is the session stickiness (or persistence). The load balancer
design must be created according to the session replication policy of the architecture.
On the other hand, the load balancer must be designed to allow the upgrade and
maintenance of the servers.

Considering the architecture explained in [Webserver architecture](http://keymon.wordpress.com/2012/01/15/some-posts-fro…job-assessment) section, the stickiness restrictions are:



	
  * Session stickiness must be set for each site. Site failure means lose of all users sessions crated in that site.

	
  * Session stickiness must be set for each farm. Server failure is allowed (session will be recovered from the session DB backend)

	
  * Session stickiness for servers into the farm is optional (Could be set to take advantage of OS disk cache).


The software that I propose is HAProxy ([http://haproxy.1wt.eu/](http://haproxy.1wt.eu/)):



	
  * Well known proven software. Big community.

	
  * Simply to configure and learn.

	
  * High performance ([http://kristianlyng.wordpress.com/2010/10/23/275k-req/](http://kristianlyng.wordpress.com/2010/10/23/275k-req/),
[http://haproxy.1wt.eu/](http://haproxy.1wt.eu/) > _Performance section_, [http://haproxy.1wt.eu/10g.html](http://haproxy.1wt.eu/10g.html))

	
  * No internal state for stickness is needed (based on cookies).
This way is easy to make it redundant (e.p. [http://www.keepalived.org/](http://www.keepalived.org/)) or
balance it with Layer 4 high-performance load-balancers.


The load balancer design consists of two layers:

	
  * One primary HAProxy LB, that will balance between sites.
Configured with session stickiness to the site using an inserted cookie, SITEID.
See primary-haproxy.conf.

	
  * One site LB in each site, balancing the site's farms.
Configured with session stickiness to the farm FARM_ID.
See site1-haproxy.conf.


[![](http://keymon.files.wordpress.com/2012/01/balancers.png)](http://keymon.files.wordpress.com/2012/01/balancers.png)

Extra comments:



	
  * Each layer can have several HAproxy instances, with the same configuration, configured with a failover solution or behind a Layer 4 load balancer. See [http://haproxy.1wt.eu/download/1.3/doc/architecture.txt](http://haproxy.1wt.eu/download/1.3/doc/architecture.txt) (Section 2) for examples.

	
  * Additionally a SSL frontend solution should be configured for SSL connections between the client
and the primary load balancer. We can use plain HTTP between balancers and servers.
I will not describe this element.

	
  * The solutions described in [HAproxy architecture documentation](http://haproxy.1wt.eu/download/1.3/doc/architecture.txt), _"4 Soft-stop for application maintenance"_, can be used.

	
  * With HAProxy 1.4 you can dynamically control servers weight.
A monitoring system can check the farms/servers health and tune the weight as needed.


This solution scales well. You simply need to add more servers, farms and sites.
Load-balancers can scale horizontally as commented.

Primary configuration:

[sourcecode launguage="bash"]
#
# Primary Load Balancer configuration: primary-haproxy.conf
#
global
	log 127.0.0.1	local0
	log 127.0.0.1	local1 notice
	#log loghost	local0 info
	maxconn 40000
	user haproxy
	group haproxy
	daemon
	#debug
	#quiet

defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
	retries	3
	option redispatch
	maxconn	2000
	contimeout	5000
	clitimeout	50000
	srvtimeout	50000

listen primary_lb_1
    # We insert cookies, add headers => http mode
    mode http

    #------------------------------------
    # Bind to all address.
    #bind 0.0.0.0:10001
    # Bind to a clusterized virtual ip
    bind 192.168.10.1:10001 transparent

    #------------------------------------
    # Cookie persistence for PHP sessions. Options
    #  - rewrite PHPSESSID: will add the server label to the session id
    #cookie	PHPSESSID rewrite indirect
    #  - insert a cookie with the identifier.
    #    Use of postonly (session created in login form) or nocache to avoid be cached
    cookie SITEID insert postonly

    # We need to know the client ip in the end servers.
    # Inserts X-Forwarded-For. Needs httpclose (no Keep-Alive).
    option forwardfor
    option httpclose

    # Roundrobin is ok for HTTP requests.
    balance	roundrobin

    # The backend sites
    # Several options are possible:
    #  inter 2000 downinter 500 rise 2 fall 5 weight 100
    server site1 192.168.11.1:10001 cookie site1 check
    server site2 192.168.11.1:10002 cookie site2 check
    # etc..
[/sourcecode]

Site configuration:

[sourcecode launguage="bash"]
#
# Site 1 load balancer configuration: syte1-haproxy.conf
#
global
log 127.0.0.1    local0
log 127.0.0.1    local1 notice
#log loghost    local0 info
maxconn 40000
user haproxy
group haproxy
daemon
#debug
#quiet

defaults
log    global
mode    http
option    httplog
option    dontlognull
retries    3
option redispatch
maxconn    2000
contimeout    5000
clitimeout    50000
srvtimeout    50000

#------------------------------------
listen site1_lb_1
grace 20000 # don't kill us until 20 seconds have elapsed

# Bind to all address.
#bind 0.0.0.0:10001
# Bind to a clusterized virtual ip
bind 192.168.11.1:10001 transparent

# Persistence.
# The webservers in same farm share the session
# with memcached. The whole site has them in a DB backend.
mode http
cookie FARMID insert postonly

# Roundrobin is ok for HTTP requests.
balance roundrobin

# Farm 1 servers
server site1ws1 192.168.21.1:80 cookie farm1 check
server site1ws2 192.168.21.2:80 cookie farm1 check
# etc...

# Farm 2 servers
server site1ws17 192.168.21.17:80 cookie farm2 check
server site1ws18 192.168.21.18:80 cookie farm2 check
server site1ws19 192.168.21.19:80 cookie farm1 check
# etc..
[/sourcecode]

