---
author: keymon
comments: true
date: 2010-08-06 09:53:43+00:00
layout: post
slug: setup-puppet-client-to-run-in-a-cron-task-with-a-random-minute
title: Setup puppet client to run in a cron task with a random minute
wordpress_id: 150
categories:
- fast-tip
- puppet
- script
- sysadmin
- trick
tags:
- cron
- efficiency
- linux
- puppet
- sysadmin
- tip
- trick
---

Puppet architecture needs a client to connect to the server to load the configuration usin a pull schema. But I do not like to have more and more daemons around and some people[ suggest avoid that](http://www.martynov.org/2009/04/running-puppet-on-big-scale.html) , so I decided to execute puppet using '--onetime' option from cron.

Obviously, I want to configure this using puppet itself. And we must ensure that the clients are executed at different times, not all at the same minute.

I searched the net and I found several aproaches to do this. There are also [feature requests.](http://projects.puppetlabs.com/issues/311)

I read somewhere that the new function [fqdn_rand() ](http://docs.puppetlabs.com/references/stable/function.html#fqdn-rand)could be used, as proposed in the feature request and posted in this [mail from Brice Figureau](http://groups.google.es/group/puppet-users/msg/2bdb4ebf21b43b32). I can not find where the hell the snippet was. At the end, I found [this pastie by Jhon Goebel](http://pastie.org/680201).

I will post my version here just to keep it wrote down.

    
     $first = fqdn_rand(30)
     $second = fqdn_rand(30) + 30
     cron { "cron.puppet.onetime":
     command => "/srv/scripts/puppet/puppet.ctl.sh onetime > /dev/null",
     user => "root",
     minute => [ $first, $second ],
     require => File["/srv/scripts/puppet/puppet.ctl.sh"],
     }
    


_… this is another random thinking from keymon ([http://keymon.wordpress.com](../))_

