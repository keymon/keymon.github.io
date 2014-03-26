---
author: keymon
comments: true
date: 2012-02-09 00:46:07+00:00
layout: post
slug: efficient-nagios-remote-monitoring-with-ssh-version-2
title: Efficient Nagios remote monitoring with SSH (version 2)
wordpress_id: 277
categories:
- fast-tip
- monitoring
- nagios
- sysadmin
- Technical
- trick
tags:
- master socket
- ssh key
---

I never liked have to install agents for different tasks like Backups or monitoring. I think that is always enough with SSH. In this post I will introduce some concepts that I am using as an alternative to the NRPE for nagios.

[Time ago I explained how to setup SSH for remote monitor servers in Nagios](http://keymon.wordpress.com/2010/04/13/using-ssh-to-monitor-remote-hosts/), using the ControlMaster feature to reuse the connection.

In that post I was using runit to keep the connections alive.

But in [OpenSSH 5.6 ](http://lwn.net/Articles/401422/) a new feature has been released:




> 
* Added a ControlPersist option to ssh_config(5) that automatically
 starts a background ssh(1) multiplex master when connecting. This
 connection can stay alive indefinitely, or can be set to
 automatically close after a user-specified duration of inactivity.




And this is COOL! We can just use some options in the [check_by_ssh plugin](http://nagiosplugins.org/man/check_by_ssh) to automatically create the session. The options are:



	
  * -i /etc/nagios/nagiosssh.id_rsa: Private ssh key generated with ssh-keygen.

	
  * -o ControlMaster=auto: Create the control master socket automatically

	
  * -o ControlPersist=yes: Enable Control persist. It will spam a ssh process in background that will keep the connection (can be stopped with -O exit)

	
  * -o ControlPath=/var/run/nagiosssh/$HOSTNAME$: Path to the control socket. We can create a dir in /var/run/nagiosssh.

	
  * -l nagiosssh -H $HOSTNAME$: User and host were we are connecting.


So, the command definition can be:

`
define command{
  command_name    check_users_ssh
  command_line    $USER1$/check_by_ssh \
    -o ControlMaster=auto \
    -o ControlPath=/var/run/nagios/$HOSTNAME$ \
    -o ControlPersist=yes \
    -i $USER6$ -H $HOSTADDRESS$ -l $USER5$ \
    'check_users -w $ARG1$ -c $ARG2$'
}
`

Note: You have to define the USER variables in resources.cfg.

Then we only need to create the proper user in the remote host. To improve the security, you can:



	
  * Use [bash in restricted mode](http://www.gnu.org/software/bash/manual/html_node/The-Restricted-Shell.html):

	
    1. Create the user 'nagiosssh' with shell=/home/nagiosssh/rbash

	
    2. Create a script /home/nagiosssh/rbash:
`#!/bin/sh
# Restricted shell for the client.
# Sets the path to checks
PATH=/home/icingassh/checks exec /bin/bash --restricted "$@"`


	
    3. Create the directory /home/icingassh/checks  and link here all the desired checks.




	
  * Restrict the ssh connection setting options in .ssh/authorized_keys. For example:
`
no-agent-forwarding,no-port-forwarding,no-pty,no-X11-forwarding,from="10.10.10.10" ssh-rsa AAAAB3NzaC1...
`





Maybe in some days I upload a chef recipe to setup this.
