---
author: keymon
comments: true
date: 2010-04-13 12:53:18+00:00
layout: post
slug: using-ssh-to-monitor-remote-hosts
title: Using SSH Control Master for efficient monitoring of remote hosts in Nagios
wordpress_id: 46
categories:
- linux/unix
- monitoring
- nagios
- script
- sysadmin
tags:
- aix
- control master
- efficiency
- linux
- monitoring
- nagios
- plugin
- runit
- shell
- ssh
- sysadmin
---

I want to use SSH to remotelly monitor some hosts. The problem with ssh  is the  overhead related to the creation of new connections.

But we can use the functionality of _Control Master_ from OpenSSH ([seehttp://www.revsys.com/writings/quicktips/ssh-faster-connections.html](//www.revsys.com/writings/quicktips/ssh-faster-connections.html)).  Using  it, ssh will connect only once reusing the connection.

This is great for monitoring scripts.

To use _Control Master_, you have to execute something like:

    
    ssh -o "ControlMaster=yes" -o "ControlPath=/somepath/ssh-%r@%h:%p" -N user@host


The idea: create a remote user in each monitored host, stablish  persistent   ssh connection and use _Control Master_ for the monitoring scripts. To deamonize and control the ssh master connections, I will use [runit](http://smarden.org/runit/)

<!-- more -->


## Implementing the idea


First, I will create the following directory structure:



	
  * SSH_CONTROL_MASTER_HOME=/etc/nagios3/ssh/: Main service  directory

	
    * ./controlpath/: Path for the _Control Master_ sockets

	
    * ./ssh-hosts-available.d/ and ./ssh-hosts-enabled.d/:  Directories for hosts entries (It will be runsv directories, see below)






    
    SSH_CONTROL_MASTER_HOME=/etc/nagios3/ssh/
    mkdir -p $SSH_CONTROL_MASTER_HOME
    mkdir -p $SSH_CONTROL_MASTER_HOME/controlpath
    mkdir -p $SSH_CONTROL_MASTER_HOME/ssh-hosts-available.d
    mkdir -p $SSH_CONTROL_MASTER_HOME/ssh-hosts-enabled.d


And we create the necesary files for the ssh connection:



	
  1. SSH keys. You can use passphrase or not, is your choice. If you  use passphrase, you should start an SSH Agent:

    
    ssh-keygen -t rsa -b 2048  -f $SSH_CONTROL_MASTER_HOME/id_rsa






	
  1. config and known_hosts files for SSH client:

    
    cat > $SSH_CONTROL_MASTER_HOME/config <<EOF
    host *
        ControlMaster auto
        IdentitiesOnly yes
        ControlPath /etc/nagios3/ssh/controlpath/ssh-%r@%h:%p
        IdentityFile /etc/nagios3/ssh/id_rsa
        UserKnownHostsFile /etc/nagios3/ssh/known_hosts
    EOF
    touch $SSH_CONTROL_MASTER_HOME/known_hosts





Next, we create a default runsv directory for the ssh services  (see [runsv manpage](http://manpages.ubuntu.com/manpages/intrepid/man8/runsv.8.html)).  What I will do is create a common run script that reads a file  called ./host.conf with diferent options.

This script depends of a ./host.conf that defines SSH_USER_HOST.  runsv ensures that pwd is ./ .




    
    mkdir $SSH_CONTROL_MASTER_HOME/basic-host/
    cat >> $SSH_CONTROL_MASTER_HOME/basic-host/run <<EOF
    <strong>#!/bin/sh
    </strong>. ./host.conf || <strong>exit</strong> 1
    
    <strong>exec</strong> sudo -u nagios ssh \
            -o <strong>"ControlMaster=yes"</strong> \
            -F /etc/nagios3/ssh/config \
            -N $SSH_USER_HOST
    EOF
    chmod +x $SSH_CONTROL_MASTER_HOME/basic-host/run





To easily create the hosts, I will use a script like following. This  script copies a "template" host in $SSH_CONTROL_MASTER_HOME/ssh-hosts-available.d and initializes the host.conf. Actually the template contains only a link to  $SSH_CONTROL_MASTER_HOME/basic-host/run




    
    <em># Create the template
    </em>mkdir $SSH_CONTROL_MASTER_HOME/ssh-hosts-available.d/template
    ln -s ../../basic-host/run $SSH_CONTROL_MASTER_HOME/ssh-hosts-available.d/template/
    
    <em># Create the script
    </em>cat > $SSH_CONTROL_MASTER_HOME/create-host.sh <<<strong>"EOF2"</strong>
    <strong>#!/bin/sh
    </strong><strong>cd</strong> $(dirname $0)
    
    <strong>if</strong> [ <strong>"$1"</strong> == <strong>""</strong> ]; <strong>then</strong>
            cat <<EOF
    Uso:
            $0 [-l] <host>
    EOF
    <strong>fi</strong>
    HOST=$1
    
    <strong>if</strong> [ -d ssh-hosts-available.d/$HOST ]; <strong>then</strong>
            <strong>echo</strong> <strong>"ssh-hosts-available.d/$HOST already exists"</strong>
            <strong>exit</strong> 1
    <strong>fi</strong>
    
    cp -Ra ssh-hosts-available.d/template  ssh-hosts-available.d/$HOST
    cat <<EOF > ssh-hosts-available.d/$HOST/host.conf
    SSH_USER_HOST=nagios@$HOST
    EOF
    <strong>echo</strong> <strong>"Execute this command in '`pwd`' to activate the host:"</strong>
    <strong>echo</strong> <strong>"ln -s ../ssh-hosts-available.d/$HOST ssh-hosts-enabled.d"</strong>
    EOF2
    chmod +x $SSH_CONTROL_MASTER_HOME/create-host.sh







## Control script


Now we need a control script to start all the _ssh services_.

This script is based on Debian init.d script skeleton. It will launch  the  command [runsvdir](http://manpages.ubuntu.com/manpages/intrepid/man8/runsvdir.8.html) on  $SSH_CONTROL_MASTER_HOME/ssh-hosts-available.d, using [start-stop-daemon](http://manpages.ubuntu.com/manpages/dapper/man8/start-stop-daemon.8.html) to _daemonize_ it.

I am not really sure if this is the best way, but it works.




    
    cat > $SSH_CONTROL_MASTER_HOME/nagios-ssh-mastercontrol.ctl.sh <<EOF
    <strong>#! /bin/sh
    </strong><em>### BEGIN INIT INFO
    </em><em># Provides:          nagios-ssh-mastercontrol
    </em><em># Required-Start:    $remote_fs
    </em><em># Required-Stop:     $remote_fs
    </em><em># Default-Start:     2 3 4 5
    </em><em># Default-Stop:      0 1 6
    </em><em># Short-Description: Nagios Master Control ssh services
    </em><em># Description:       Starts a set of connections via SSH to servers stablishing mastercontrol
    </em><em>#                    connections to be used in nagios monitoring
    </em><em>### END INIT INFO
    </em>
    <em># Author: Hector Rivas Gandara <keymon@gmail.com>
    </em>
    <em># Do NOT "set -e"
    </em>
    <em># PATH should only include /usr/* if it runs after the mountnfs.sh script
    </em>SVDIR=<strong>"/etc/nagios3/ssh/ssh-hosts-enabled.d/"</strong>
    
    <strong>PATH</strong>=/sbin:/usr/sbin:/bin:/usr/bin
    DESC=<strong>"Nagios SSH Master Control connections"</strong>
    NAME=nagios-ssh-mastercontrol
    DAEMON=/usr/bin/runsvdir
    DAEMON_ARGS=$SVDIR
    PIDFILE=/var/run/$NAME.pid
    SCRIPTNAME=/etc/init.d/$NAME
    USER=nagios
    
    <em># Exit if the package is not installed
    </em><em>#[ -x "$DAEMON" ] || exit 0
    </em>
    <em># Read configuration variable file if it is present
    </em>[ -r /etc/default/$NAME ] && . /etc/default/$NAME
    
    <em># Load the VERBOSE setting and other rcS variables
    </em>. /lib/init/vars.sh
    
    <em># Define LSB log_* functions.
    </em><em># Depend on lsb-base (>= 3.0-6) to ensure that this file is present.
    </em>. /lib/lsb/init-functions
    
    <em>#
    </em><em># Function that starts the daemon/service
    </em><em>#
    </em>do_start()
    {
            <em># Return
    </em>        <em>#   0 if daemon has been started
    </em>        <em>#   1 if daemon was already running
    </em>        <em>#   2 if daemon could not be started
    </em>        start-stop-daemon -u $USER -u $USER --start  --pidfile $PIDFILE --<strong>exec</strong> $DAEMON --<strong>test</strong> > /dev/null \
                    || <strong>return</strong> 1
            start-stop-daemon -b -m -u $USER --start --pidfile $PIDFILE --<strong>exec</strong> $DAEMON -- \
                    $DAEMON_ARGS \
                    || <strong>return</strong> 2
            <em># Add code here, if necessary, that waits for the process to be ready
    </em>        <em># to handle requests from services started subsequently which depend
    </em>        <em># on this one.  As a last resort, sleep for some time.
    </em>}
    
    <em>#
    </em><em># Function that stops the daemon/service
    </em><em>#
    </em>do_stop()
    {
    <em>#       for i in $SVDIR/*; do
    </em><em>#               if [ -d $i ]; then
    </em><em>#                       echo "Stopping $i..."
    </em><em>#                       sv down $i
    </em><em>#               fi
    </em><em>#       done
    </em>        <em># Return
    </em>        <em>#   0 if daemon has been stopped
    </em>        <em>#   1 if daemon was already stopped
    </em>        <em>#   2 if daemon could not be stopped
    </em>        <em>#   other if a failure occurred
    </em>        start-stop-daemon --stop  --retry=HUP/30/TERM/5 --pidfile $PIDFILE
            RETVAL=<strong>"$?"</strong>
            [ <strong>"$RETVAL"</strong> = 2 ] && <strong>return</strong> 2
            <em># Wait for children to finish too if this is a daemon that forks
    </em>        <em># and if the daemon is only ever run from this initscript.
    </em>        <em># If the above conditions are not satisfied then add some other code
    </em>        <em># that waits for the process to drop all resources that could be
    </em>        <em># needed by services started subsequently.  A last resort is to
    </em>        <em># sleep for some time.
    </em>        start-stop-daemon --stop --quiet --oknodo --retry=0/30/KILL/5 --pidfile $PIDFILE
            [ <strong>"$?"</strong> = 2 ] && <strong>return</strong> 2
            <em># Many daemons don't delete their pidfiles when they exit.
    </em>        <em>#rm -f $PIDFILE
    </em>        <strong>return</strong> <strong>"$RETVAL"</strong>
    }
    
    <em>#
    </em><em># Function that sends a SIGHUP to the daemon/service
    </em><em>#
    </em>do_reload() {
            <em>#
    </em>        <em># If the daemon can reload its configuration without
    </em>        <em># restarting (for example, when it is sent a SIGHUP),
    </em>        <em># then implement that here.
    </em>        <em>#
    </em>        start-stop-daemon -u $USER -m --stop --signal 1 --quiet --pidfile $PIDFILE --name $NAME
            <strong>return</strong> 0
    }
    
    <strong>case</strong> <strong>"$1"</strong> <strong>in</strong>
      start)
            [ <strong>"$VERBOSE"</strong> != no ] && log_daemon_msg <strong>"Starting $DESC"</strong> <strong>"$NAME"</strong>
            do_start
            <strong>case</strong> <strong>"$?"</strong> <strong>in</strong>
                    0|1) [ <strong>"$VERBOSE"</strong> != no ] && log_end_msg 0 ;;
                    2) [ <strong>"$VERBOSE"</strong> != no ] && log_end_msg 1 ;;
            <strong>esac</strong>
            ;;
      stop)
            [ <strong>"$VERBOSE"</strong> != no ] && log_daemon_msg <strong>"Stopping $DESC"</strong> <strong>"$NAME"</strong>
            do_stop
            <strong>case</strong> <strong>"$?"</strong> <strong>in</strong>
                    0|1) [ <strong>"$VERBOSE"</strong> != no ] && log_end_msg 0 ;;
                    2) [ <strong>"$VERBOSE"</strong> != no ] && log_end_msg 1 ;;
            <strong>esac</strong>
            ;;
      <em>#reload|force-reload)
    </em>        <em>#
    </em>        <em># If do_reload() is not implemented then leave this commented out
    </em>        <em># and leave 'force-reload' as an alias for 'restart'.
    </em>        <em>#
    </em>        <em>#log_daemon_msg "Reloading $DESC" "$NAME"
    </em>        <em>#do_reload
    </em>        <em>#log_end_msg $?
    </em>        <em>#;;
    </em>  restart|force-reload)
            <em>#
    </em>        <em># If the "reload" option is implemented then remove the
    </em>        <em># 'force-reload' alias
    </em>        <em>#
    </em>        log_daemon_msg <strong>"Restarting $DESC"</strong> <strong>"$NAME"</strong>
            do_stop
            <strong>case</strong> <strong>"$?"</strong> <strong>in</strong>
              0|1)
                    do_start
                    <strong>case</strong> <strong>"$?"</strong> <strong>in</strong>
                            0) log_end_msg 0 ;;
                            1) log_end_msg 1 ;; <em># Old process is still running
    </em>                        *) log_end_msg 1 ;; <em># Failed to start
    </em>                <strong>esac</strong>
                    ;;
              *)
                    <em># Failed to stop
    </em>                log_end_msg 1
                    ;;
            <strong>esac</strong>
            ;;
      *)
            <em>#echo "Usage: $SCRIPTNAME {start|stop|restart|reload|force-reload}" >&2
    </em>        <strong>echo</strong> <strong>"Usage: $SCRIPTNAME {start|stop|restart|force-reload}"</strong> >&2
            <strong>exit</strong> 3
            ;;
    <strong>esac</strong>
    
    :
    EOF
    chmod +x $SSH_CONTROL_MASTER_HOME/nagios-ssh-mastercontrol.ctl.sh







## Defining a new service in Nagios  via SSH


Finally, we use the nagios plugin  [check_by_ssh](http://nagiosplugins.org/man/check_by_ssh) to remotely execute our plugins,  adding the options needed to use ControlMaster:



	
  * -o "ControlMaster=no": This will use the ControlMaster, but not create it if it does not  exists.

	
  * -o "ControlPath=/etc/nagios3/ssh/controlpath/ssh-%r@%h:%p"

	
  * -o "PasswordAuthentication=no"


For example:

    
    define command{
            command_name    check_myplugin_by_ssh
            command_line    /usr/lib/nagios/plugins/check_by_ssh -H $HOSTADDRESS$ -l nagiosusr  -o "ControlMaster=no" -o "ControlPath=/etc/nagios3/ssh/controlpath/ssh-%r@%h:%p" -o "PasswordAuthentication=no" -C "/a_path/myplugin $ARG2$"
            }




## Defining new hosts


First, we should create the new user that will execute nagios plugins in  the  remote server, and populate its $HOME/.ssh/authorized_keys with   $SSH_CONTROL_MASTER_HOME/id_rsa.pub. For instance, we can do:

    
    ssh-copy-id -i $SSH_CONTROL_MASTER_HOME/id_rsa.pub nagiosuser@machine


Next, to define a new host, we use the script create-host.sh

    
    # ./create-host.sh ahost
    Execute this command in '/etc/nagios3/ssh' to activate the host:
    ln -s ../ssh-hosts-available.d/ahost ssh-hosts-enabled.d


With runsvdir, the service will start automacly if we create  the link, as documented:

_ At least every five seconds runsvdir checks whether the  time   of  last _


> modification,  the  inode, or the device, of the services directory dir has changed.  If so, it re-scans the service directory, and if it  sees a  new subdirectory, or new symlink to a directory, in dir, it starts a new runsv(8) process;_ _


But we should firstly accept the ssh host key of the client. We can do  it by simply executing the run script:

    
    # cd $SSH_CONTROL_MASTER_HOME/ssh-hosts-enabled.d/ecvignevcml1/
    # sudo -u nagios ./run
    The authenticity of host 'ecvignevcml1 (172.16.13.14)' can't be established.
    RSA key fingerprint is 28:6d:eb:38:42:e9:82:90:22:6e:af:17:ad:86:44:83.
    Are you sure you want to continue connecting (yes/no)? yes
    <Ctrl+C>


And, now, we can link it:

    
    ln -s ../ssh-hosts-available.d/ahost ssh-hosts-enabled.d
    <h3 id="Example:DiskusagemonitoringinAIX">Example: Disk usage monitoring  in AIX</h3>
    I will use the technique described in this article with the <a href="../2010/04/15/restrict-a-set-of-remote-commands-to-execute-with-sshbashsudo/">restricted shell functionality in bash</a>, to  implement the disk usage monitoring in a remote host with AIX.
    We will use this data:
    <ul>
    	<li>Monitoring user name: <tt>monxusr</tt></li>
    	<li>User home: <tt>/srv/mon/monxusr</tt></li>
    	<li>User groups: <tt>staff,sshcon</tt> (ssh restricts the  connection to sshcon group)</li>
    </ul>
    <h4 id="Installingtheplugins">First of all, we install the plugins: we will compile the nagios plugins in AIX, and install them with  stow. I will not comment this:</h4>
    



    
    tar -xvzf nagios-plugins_1.4.11.orig.tar.gz
    cd nagios-plugins-1.4.11
    ./configure --help
    ./configure --prefix=/usr/local/stow/nagios-plugins-1.4.11
    sudo make install
    cd /usr/local/stow
    sudo mkdir -p /usr/local/share/locale
    sudo stow nagios-plugins-1.4.11/


In the remote host we create the user, without password. The shell will  be initally the default.

    
    RESTRICTED_USER=monxusr
    USERHOME=/srv/mon/monxusr
    GROUPS=staff,sshcon 
    
    mkdir $(dirname $USERHOME)
    mkuser -R compat id=10102 pgrp=staff groups=staff,sshcon home=$USERHOME maxexpired=-1 maxexpired=-1 loginretries=-1  $RESTRICTED_USER
    pwdadm -R compat -c $RESTRICTED_USER


We create the restricted shell script, we add it to valid shells and we  change  the user to use this shell.

    
    mkdir $USERHOME/bin
    cat >$USERHOME/bin/rbash <<EOF
    #!/usr/bin/bash -e
    export PATH=$USERHOME/bin
    f=\$1
    if [ "\$1" != "" ]; then
     shift
     exec /bin/bash \$f "\$*"
    else
     exec /bin/bash \$*
    fi
    EOF
    chmod +x $USERHOME/bin/rbash
    
    chsec -f /etc/security/login.cfg -s usw -a shells=$(lssec -f /etc/security/login.cfg -s usw -a shells | cut -f 2 -d =),$USERHOME/bin/rbash
    chuser -R compat shell=$USERHOME/bin/rbash  $RESTRICTED_USER


We configure ssh keys:

    
    sudo -u $RESTRICTED_USER mkdir $USERHOME/.ssh
    sudo -u $RESTRICTED_USER tee -a $USERHOME/.ssh/authorized_keys <<EOF
    ssh-rsa AAAAqwertyuiopasdfghjklzxcvbnmqwertyuiopasdfghjklzxcvbnmqwertyuiopasdfghjklzxcvbnmqwertyuiopasdfghjklzxcvbnmqwertyuiopasdfghjklzxcvbnmqwertyuiopasdfghjklzxcvbnmqwertyuiopasdfghjklzxcvbnmqwertyuiopasdfghjklzxcvbnm nagios@nagiosserver
    EOF


And finally we link the plugins to the nagios plugins:

    
    ln -s /usr/local/libexec/* $USERHOME/bin


We cant test it:

    
    # su - monxusr
    $  check_disk -w 10% -c 5% -p /tmp -p /var -C -w 100000 -c 50000 -p /
    DISK CRITICAL - free space: /tmp 913 MB (89% inode=99%); /var 138 MB (22% inode=84%); / 17 MB (6% inode=59%);| /tmp=110MB;921;972;0;1024 /var=469MB;547;577;0;608 /=254MB;-99728;-49728;0;272




#### To set the nagios server, first we  configure the ssh connection as described:


# SSH_CONTROL_MASTER_HOME=/etc/nagios3/ssh/

    
    # cd $SSH_CONTROL_MASTER_HOME
    # ./create-host.sh remoteserver
    Execute this command in '/etc/nagios3/ssh' to activate the host:
    
    # (cd ./ssh-hosts-enabled.d/remoteserver/; ./run)
    The authenticity of host 'remoteserver (192.168.1.2)' can not be established.
    RSA key fingerprint is 14:0c:5e:20:0c:22:34:58:e3:da:06:55:fd:e5:58:4e.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added 'remoteserver,192.168.1.2' (RSA) to the list of known hosts.
    <Ctrl+C>
    
    # ln -s ../ssh-hosts-available.d/remoteserver ssh-hosts-enabled.d
    
    # ps -fea| grep ssh |grep remoteserver
    nagios   29674 29143  0 17:07 ?        00:00:00 ssh -o ControlMaster=yes -F /etc/nagios3/ssh/config -N monxusr@remoteserver


And  we test the plugin:

    
    # time ssh -F /etc/nagios3/ssh/config monxusr@remoteserver check_disk -w 10% -c 5% -p /tmp -p /var -C -w 100000 -c 50000 -p /
    DISK CRITICAL - free space: /tmp 913 MB (89% inode=99%); /var 138 MB (22% inode=84%); / 17 MB (6% inode=58%);| /tmp=110MB;921;972;0;1024 /var=469MB;547;577;0;608 /=254MB;-99728;-49728;0;272
    
    real    0m0.068s
    user    0m0.008s
    sys     0m0.004s


As you see is quite fast if you compare it without ControlMaster:

root@dclinuxapps1:/etc/nagios3/ssh/# time ssh -o ControlMaster=no -i id_rsa -o UserKnownHostsFile=/etc/nagios3/ssh/known_hosts monxusr@xcaixnetiml1 check_disk -w 10% -c 5% -p /tmp -p /var -C -w 100000 -c 50000 -p /

    
    DISK CRITICAL - free space: /tmp 913 MB (89% inode=99%); /var 138 MB (22% inode=84%); / 17 MB (6% inode=58%);| /tmp=110MB;921;972;0;1024 /var=469MB;547;577;0;608 /=254MB;-99728;-49728;0;272
    
    real    0m0.581s
    user    0m0.016s
    sys     0m0.004s


Finally we define the nagios command and a service for the server:

    
    define command{
            command_name    check_disk_by_ssh
            command_line    /usr/lib/nagios/plugins/check_by_ssh -H $HOSTADDRESS$ -l monxusr -o "ControlMaster=no" -o "ControlPath=/etc/nagios3/ssh/controlpath/ssh-%r@%h:%p" -o "PasswordAuthentication=no" -C "check_disk -w '$ARG1$' -c '$ARG2$' -e -p '$ARG3$'"
            }
    
    define service {
            host_name                       remoteserver
            service_description             remoteserver disk usage
            display_name                    remoteserver disk usage
            check_command                   check_disk_by_ssh!devel!10!5!/
            use                             generic-service
    }
