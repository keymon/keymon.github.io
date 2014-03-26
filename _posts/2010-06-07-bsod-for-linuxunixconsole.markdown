---
author: keymon
comments: true
date: 2010-06-07 23:36:55+00:00
layout: post
slug: bsod-for-linuxunixconsole
title: BSoD for linux/unix/console
wordpress_id: 111
categories:
- fast-tip
- humor
- Personal
- script
tags:
- bash
- bsod
- console
- linux
---

I do not remember where I got this, but it is fun. I will put it in the motd of my hosts on April Fools' day.

This script will display on a console a windows like BSoD (Blue Script of Day):

[![                                    Linux ws         An exception 0E has occurred at 0028:C0018DBA in VxD IFSMGR(01) +        0000340A.  This was called from 0028:C0034118 in VxD NDIS(01) +        00000D7C.  It may be possible to continue normally.         *  Press any key to attempt to continue        *  Press CTRL+ALT+DEL to restart your computer. You will           lose any unsaved information in all applications                             Press Any key to continue.](http://keymon.files.wordpress.com/2010/06/linux-bsod.jpg?w=300)](http://keymon.files.wordpress.com/2010/06/linux-bsod.jpg)<!-- more -->

The original script used the ESC (033 in octal)  character  directly. I use '@' that I replace for ESC so it is more readable and easy to handle.

[sourcecode language="bash"]
#!/bin/sh
cat << EOM | tr '@' '\033'
@[0m@[2J@[1;1H@[0;25;37;44m
EOM
clear
cat << EOM | tr '@' '\033'

                                   @[34;47m Linux @[44mws@[37m

@[1m       An exception 0E has occurred at 0028:C0018DBA in VxD IFSMGR(01) +
       0000340A.  This was called from 0028:C0034118 in VxD NDIS(01) +
       00000D7C.  It may be possible to continue normally.

       *  Press any key to attempt to continue
       *  Press CTRL+ALT+DEL to restart your computer. You will
          lose any unsaved information in all applications

                           Press Any key to continue.

                                                                                @[0m
EOM
read
[/sourcecode]
