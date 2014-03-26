---
author: keymon
comments: true
date: 2011-09-22 10:05:10+00:00
layout: post
slug: script-to-determine-the-login-user-of-a-process
title: Script to determine the login user of a process
wordpress_id: 261
categories:
- script
- sysadmin
---

Sometimes you need to know who was the user that did login in a linux/unix server, but after several "sudo" or "su" commands (and others programs that change the permissions) you have lost the information.

You can try to determine the user using the tty of the process tree, querying the process parents.

With this idea, I wrote this small script: `whowasi.sh`

[sourcecode language="bash"]
#!/bin/env bash
# This scripts allows  determine the user used to login in the
# machine to run the given process.
# 
SCRIPT_NAME=$0

# Command names to be considered as login commands
LOGIN_PROGRAMS="sshd telnetd login" 

# Get all pids of the parents of a pid
get_parent_pids() {
    echo $1
    ppid=$(ps -o ppid -p $1 | awk '/[0-9]+/ {print $1}' )
    [ $ppid == 1 -o $ppid == 0 ] && return
    get_parent_pids $ppid
}

# Get users of parent process of a pid
get_parent_users() {
	get_parent_pids $1 | xargs -n1 ps -o user= -p | uniq | awk '{print $1}'
}

get_parent_users_commands() {
	get_parent_pids $1 | xargs -n1 ps -o user= -o comm= -p | uniq
}

get_parent_users_ttys() {
	get_parent_pids $1 | xargs -n1 ps -o user= -o tty= -p | uniq
}


get_firstuser_after_login() {
	cmd="egrep -B1 -m1" # Get the line before, and stop on first match
	for p in $LOGIN_PROGRAMS; do 
		cmd="$cmd -e '^(.*/)?$p\$'" 
	done
	get_parent_users_commands $1 | eval $cmd | awk '{ print $1; exit; }'
}

get_firstuser_after_root() {
	get_parent_users $1 | grep -B1 -m1 root | awk '{print $1;exit;}'
}

get_firstuser_with_tty() {
	get_parent_users_ttys $1 | grep -B1 -m1 \?  | awk '{print $1;exit;}'
}


print_help() {
	cat <<EOF
Usage $SCRIPT_NAME [Option...] [pid]

Prints the users that where used to start a process.

By default it will use the current process.
	
Options
	-h:		This help.
	-t:		Print the user of the first process having a valid tty (not ?) 
			This is the default behaviour.
	-a:		Print all processes.
	-r:		Print only user started after the first root (usually the one that login in)
	-l:		Print the user after a login program ($LOGIN_PROGRAMS)
	        Requires GNU egrep.
EOF
}

mode=tty
while true; do
	case $1 in
		"")
			break
		;;
		"-a")
			mode=all
		;;
		"-l")
			mode=login
		;; 
		"-r")
			mode=root
		;;
		"-t")
			mode=tty
		;;
		"-h")
			printhelp
			exit
		;;
		"-*")
			echo "$SCRIPT_NAME: Unknown option '$1'"
			printhelp
			exit
		;;
		*)
			args="$args $1"
		;;
	esac
	shift
done
set -- $args

pid=${1:-$$}

if ! ps -p $pid >/dev/null 2>&1; then
	echo "$SCRIPT_NAME: Unable to find process '$pid'"
	exit 1
fi

case $mode in 
	all)
		get_parent_users $pid
	;;
	login)
		get_firstuser_after_login $pid
	;;
	root)
		get_firstuser_after_root $pid
	;;
	tty)
		get_firstuser_with_tty $pid
	;;
esac
[/sourcecode]
