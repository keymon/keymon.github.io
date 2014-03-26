---
author: keymon
comments: true
date: 2012-01-15 04:10:39+00:00
layout: post
slug: routing-script
title: 'New job assessment: Routing Script'
wordpress_id: 293
categories:
- linux/unix
- Technical
- trick
---

These is one of the proposed solutions for the job assessment [commented in a previous post](http://keymon.wordpress.com/2012/01/15/some-posts-fro…job-assessment).


> _Provide a script manipulating the routing table to send all outgoing traffic originating from ipaddress: 85.14.228.248 through gw 85.14.228.254 and all other traffic through gateway 10.12.0.254_


One basically have to execute these commands:

[sourcecode language="bash"]
# Default route
ip route del default table 254
ip route add default via 192.168.1.1 dev wlan0 table 254

# alternative route and its rule
ip route del default table 1
ip route add default via 85.14.228.254 dev wlan0 table 1
ip rule del from 85.14.228.248
ip rule add from 85.14.228.248 table 1
[/sourcecode]

I delete the previous default route and rule to ensure that the commands will be a success and will update the configuration.

A more convenient script could be:

[sourcecode language="bash"]
#!/bin/bash

# Default route
DEFAULT_ROUTE=10.12.0.254
DEFAULT_DEV=eth0

# Create the diferent routes, where
#  NAME[id]  = name of routing table (for documentation purposes)
#  ROUTE[id] = destination
#  SRCS[id]  = list of routed ips
#  DEV[id]     = network device
#  id          = number of routing table (1..253)
#
NAME[1]=uplink1
ROUTE[1]="85.14.228.254"
DEV[1]=eth1
SRCS[1]="85.14.228.248"

#-----------------------------------------
# Set the "main" table
NAME[254]=main
ROUTE[254]=$DEFAULT_ROUTE
DEV[254]=$DEFAULT_DEV

# debug
#ip() { echo "> ip $*"; command ip $*; }

for i in {255..1}; do
[ ! -z "${ROUTE[$i]}" ] || continue

# Delete default route if exists
ip route list table $i | grep -q default && \
echo "Deleting default entry for route table ${NAME[$i]}($i)..." && \
ip route del default table $i

# Create the new table default route
echo "Creating route table '${NAME[$i]}($i)' with default via gw ${ROUTE[$i]}"
ip route add default via "${ROUTE[$i]}" dev ${DEV[$i]} table $i || continue

# Create
for ip in ${SRCS[i]}; do
# Delete rule if exists
ip rule list |grep -q "from $ip" && \
echo " - deleting rule from $ip..." && \
ip rule del from $ip

# Add the source rule
echo " + adding rule from $ip..."
ip rule add from $ip table $i
done
done

[/sourcecode]
