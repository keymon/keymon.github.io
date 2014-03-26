---
author: keymon
comments: true
date: 2010-10-21 11:06:05+00:00
layout: post
slug: aix-vs-gnu-core-utils-tools-the-dd-command-with-skip-parameter
title: 'AIX vs GNU core utils tools: the ''dd'' command with skip parameter. '
wordpress_id: 224
categories:
- aix
- Misc
- sysadmin
tags:
- aix
- efficiency
- sysadmin
- trick
---

Any Linux & Unix admin knowns this fact: **GNU tools are MUCH MORE  better tools than AIX, BSD, Solaris or HP-UX tools.**

GNU tools have much less bugs, much more functionality and options,  localization, better documentation, they are standard, most of the  scripts are built based on GNU tools, etc, etc,etc. Why the hell they do  not throw out their ugly-buggy-limitated tools and install the GNU  tools in their systems by default???

Here you have an example of a weird behaviour in the 'dd' command in the AIX  platform: With the skip=<Num. blocks> parameter the 'dd' command skips  the blocks, but **it actually reads them** (no matter if you are working on a filesystem with file  random access). So, if you are  working with big files (in my case, 50GB) you have to read ALL the  blocks in memory before access the requested position. That means huge  I/O, usage of memory in cache, etc...

IBM guys: you do not know that there is a [lseek(2)](http://linux.die.net/man/2/lseek) function?

Here you have an example of the time that takes read 2MB from a big  file, skiping 1000MB. Using native 'dd' command takes 12s:

    
    $ time /usr/bin/dd if=a_big_big_file.data skip=1000 bs=1M count=2 of=/dev/null
    2+0 records in.
    2+0 records out.
    
    real    0m12.059s
    user    0m0.013s
    sys     0m1.419s


With GNU's version, less than a second:

    
    $ time /opt/freeware/bin/dd if=a_big_big_file.data skip=1000 bs=1M count=2 of=/dev/null
    2+0 records in
    2+0 records out
    
    real    0m0.024s
    user    0m0.002s
    sys     0m0.006s


Note: You can find the GNU's dd tool in [AIX Linux ToolBox](http://www-03.ibm.com/systems/power/software/aix/linux/toolbox/download.html) coreutils package.

_**Update: **_I contacted the IBM support and they told me that using the option conv=iblock ,"dd" will behave as expected. But IMHO the [documentation](http://publib.boulder.ibm.com/infocenter/pseries/v5r3/index.jsp?topic=/com.ibm.aix.cmds/doc/aixcmds2/dd.htm) does not explicitily say that:

_**iblock**,** oblock**_
    _Minimize data loss resulting from a read or write error on direct access devices. If you specify the **iblock** variable and an error occurs during a block read (where the block size is 512 or the size specified by the[**ibs=**](http://publib.boulder.ibm.com/infocenter/pseries/v5r3/topic/com.ibm.aix.cmds/doc/aixcmds2/dd.htm#a138922db)InputBlockSize variable), the **dd** command attempts to reread the data block in smaller size units. If the **dd** command can determine the sector size of the input device, it reads the damaged block one sector at a time. Otherwise, it reads it 512 bytes at a time. The input block size (**ibs**) must be a multiple of this retry size. This option contains data loss associated with a read error to a single sector. The** oblock** conversion works similarly on output._
