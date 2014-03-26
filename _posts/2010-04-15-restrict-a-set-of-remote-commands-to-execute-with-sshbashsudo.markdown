---
author: keymon
comments: true
date: 2010-04-15 17:19:08+00:00
excerpt: "Sometimes we need to allow some users to remotelly execute commands in a\
  \ server via ssh, but we want to restrict the commands to execute. There are some\
  \ solutions around, like restricted shell or wrappers, but we can implement a simply\
  \ solution using bash and sudo.\n\nThe idea is to use the restricted shell functionality\
  \ in bash. We will simply:\n\n    * Create a new user for this prupose\n    * Asign\
  \ it a script that will restrict the $PATH variable\n    * Link there the needed\
  \ commands (or create scripts that call sudo)\n    * Set asymetric keys in ssh.\
  \ \n\nHere you have all commands for Linux or AIX, but it will work anyway (except\
  \ the creation of the user)."
layout: post
slug: restrict-a-set-of-remote-commands-to-execute-with-sshbashsudo
title: Restrict a set of remote commands to execute with ssh+bash+sudo
wordpress_id: 48
categories:
- linux/unix
- script
- trick
tags:
- aix
- bash
- linux
- path
- restricted shell
- scripting
- security
- ssh
- sudo
- sysadmin
- trick
---



Sometimes we need to allow some users to remotelly execute commands in a  server via ssh,  but we want to restrict the commands to execute. There are some  solutions around,  like restricted shell or wrappers, but we can implement a simply  solution using bash and sudo.

The idea is to use the [restricted shell functionality in bash](http://www.gnu.org/software/bash/manual/bashref.html#The-Restricted-Shell). We will  simply:



	
  * Create a new user for this prupose

	
  * Asign it a script that will restrict the $PATH variable

	
  * Link there the needed commands (or create scripts that call  sudo)

	
  * Set asymetric keys in ssh.


Here you have all commands for Linux or AIX, but it will work anyway  (except the creation of the user).




    
    <em># You have to set this variables before execute any command
    </em><em># ---------------------------------------------------
    </em><em># If we will use sudo for commands or only links
    </em>USE_SUDO=yes
    <em># User restricted and commands to link or sudo
    </em>RESTRICTED_USER=jailuser
    RESTRICTED_COMMANDS=<strong>"
    /somepath/somecommand1
    /somepath/somecommand1
    /somepath/somecommand1
    "</strong>
    <em># Remote host where execute commands
    </em>REMOTE_HOST=ahost
    <em># final user to execute commands (sudo)
    </em>DEST_USER=arealuser
    <em># ---------------------------------------------------
    </em>


_
_
**case** `uname` **in**
AIX)
_# Add shell script as new user
_ chsec -f /etc/security/login.cfg -s usw -a shells=$(lssec -f /etc/security/login.cfg -s usw -a shells | cut -f 2 -d =),/home/$RESTRICTED_USER/bin/rbash
mkdir /home/$RESTRICTED_USER/bin

_# Create user
_ mkuser groups=sshcon maxexpired=-1 loginretries=-1 $RESTRICTED_USER

_# Do not need to change password
_ pwdadm -c $RESTRICTED_USER
;;
_# At this moment only debian
_ Linux)
adduser --shell /home/$RESTRICTED_USER/bin/rbash --disabled-password  --no-create-home $RESTRICTED_USER
;;
**esac**

_# Create shell script for restricted mode
_cat >/home/$RESTRICTED_USER/bin/rbash <<EOF
**#!/usr/bin/bash -e
****export** **PATH**=/home/$RESTRICTED_USER/bin
f=\$1
**if** [ **"\$1"** != **""** ]; **then**
**shift**
**exec** /bin/bash \$f **"\$*"**
**else**
**exec** /bin/bash \$*
**fi**
EOF
chmod +x /home/$RESTRICTED_USER/bin/rbash

_# Configure the commands
_**if** [ **"$USE_SUDO"** == **"yes"** ]
_# Sudoers
_ **for** i **in** $RESTRICTED_COMMANDS; **do**
[ **"$sudocmd"** ] || sudocmd=**"NOPASSWD:$i"** && sudocmd=**"$sudocmd,NOPASSWD:$i"**
**done**
sudocmd=**"$RESTRICTED_USER ALL=($DEST_USER) $sudocmd"**
**echo** **"Add this line to /etc/sudoers: '$sudocmd'"**

**for** i **in** $RESTRICTED_COMMANDS; **do**
cmdfile=$(basename $i)
cat > /home/$RESTRICTED_USER/bin/$cmdfile <<EOF
**#!/bin/sh
****exec** sudo -u $DEST_USER $i \$@
EOF
chmod +x /home/$RESTRICTED_USER/bin/$cmdfile
**done**

**else**
_# Link commands
_ ln -sf $RESTRICTED_COMMANDS /home/$RESTRICTED_USER/bin/
**fi**




Optionally, in origin server, we create a key and the adapters commands.  We can create a common script and link to it the other commands.




    
    <em># Create key
    </em>ssh-keygen -i rsa_id
    
    <em># Define commands
    </em>cat > .$RESTRICTED_USER.$REMOTE_HOST.cmd <<EOF
    <strong>#!/bin/sh
    </strong>ssh -T -o IdentitiesOnly yes -o StrictHostKeyChecking=no -i id_rsa \
        $RESTRICTED_USER@$REMOTE_HOST \$(basename \$0) \$@
    EOF
    <strong>for</strong> i <strong>in</strong> $RESTRICTED_COMMANDS; <strong>do</strong>
        cmdfile=$(basename $i)
        ln -s .$RESTRICTED_USER.$REMOTE_HOST.cmd $cmdfile
    <strong>done</strong>
    





Finally we add the public key in destination server




    
    mkdir /home/$RESTRICTED_USER/.ssh
    cat > /home/$RESTRICTED_USER/.ssh/authorized_keys <<EOF
    <here goes your public key id_rsa.pub>
    EOF
    chown -R $RESTRICTED_USER /home/$RESTRICTED_USER/.ssh





Easy, isn't it?


