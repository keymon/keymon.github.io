---
author: keymon
comments: true
date: 2011-09-22 12:10:32+00:00
layout: post
slug: doit-launch-programs-on-your-windows-desktop-from-your-unix-servers
title: 'Doit: Launch programs on your windows desktop from your Unix servers'
wordpress_id: 265
categories:
- sysadmin
- Technical
- trick
---

For the last years I had the same problem: I was running windows as desktop and managing Linux/Unix?. Of cuorse, to minimize the pain, I use Cygwin and/or colinux, that make my life easier.

Often I need to open files remotely, but is so tedious to find them in the samba share... and then I've found this tool: doit [ http://www.chiark.greenend.org.uk/~sgtatham/doit/](http://www.chiark.greenend.org.uk/%7Esgtatham/doit/), from Simon Tatham, the putty author.

It allows execute commands in your box from the remote server, automatically translating the paths (in case you are using samba).


### Fast installation





	
  1. Client on Unix side (1):



	
  1. Download and compile:
[sourcecode language="bash"]
curl http://www.chiark.greenend.org.uk/~sgtatham/doit/doit.tar.gz | tar -xvzf -
cd doit
cc -o doitclient doitclient.c doitlib.c -lsocket -lnsl -lresolv
[/sourcecode]


	
  2. Install. I use [ stow](http://www.gnu.org/software/stow/manual.html)for my adhoc binaries:

[sourcecode language="bash"]
##  Preset variables.
LOCAL_BINARIES=~/local
PLATFORM="$(uname -s)-$(uname -p)"
PATH=$PATH:$LOCAL_BINARIES/$PLATFORM/bin

STOW_HOME=$LOCAL_BINARIES/$PLATFORM/stow

mkdir -p $STOW_HOME/doit/bin
cp doitclient $STOW_HOME/doit/bin
for i in wf win winwait wcmd wclip www wpath; do
 ln -s doitclient $STOW_HOME/doit/bin/$i
done

cd $STOW_HOME
stow doit
[/sourcecode]





	
  2. Shared secret setup and configuration:
[sourcecode language="bash"]
dd if=/dev/random of=$HOME/.doit-secret bs=64 count=1
chmod 640 $HOME/.doit-secret 
echo "secret $HOME/.doit-secret" &gt; $HOME/.doitrc
[/sourcecode]

Then set the mappings as described in the documentation. For instance:

    
    host
      map /home/ \\sambaserver\






	
  3. If you are using su (or sudo reseting passwords), you will lose the SSH_CLIENT variable. But can set the $DOIT_HOST variable. You can use this:
[sourcecode language="bash"]
cat <<"EOF" >> ~/.bashrc
# DOIT_HOST variable, for the DoIt tool (Integration with windows desktops)
export DOIT_HOST=$(who -m | sed 's/.*(\(.*\)).*/\1/')
EOF
[/sourcecode]


	
  4. Setup the client on a windows box. You can copy the .doit-secret or use samba to access to your home. 
Just create a link to "doit.exe secret.file", for instance:

    
    \\sambaserver\keymon\local\Linux-x86\stow\doit\doit.exe \\sambaserver\keymon\.doit-secret










### Conclusions


It is really cool, and it really works.

My only concern is that is the key, that should be shared. One solution can be use environment vars or event the Putty ‘Answerback to ^E’ ([ http://tartarus.org/~simon/putty-snapshots/htmldoc/Chapter4.html#config-answerback](http://tartarus.org/%7Esimon/putty-snapshots/htmldoc/Chapter4.html#config-answerback)), but I am not sure how implement it.

(1) On solaris, compiling with GCC, I got this error:

    
    /var/tmp//cc5ZYGYW.o: In function `main':
    doitclient.c:(.text+0x29e8): undefined reference to `hstrerror'
    collect2: ld returned 1 exit status


This is solved solved adding -lsocket.

