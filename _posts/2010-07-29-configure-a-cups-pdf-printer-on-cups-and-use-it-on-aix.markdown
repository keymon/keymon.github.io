---
author: keymon
comments: true
date: 2010-07-29 11:17:16+00:00
layout: post
slug: configure-a-cups-pdf-printer-on-cups-and-use-it-on-aix
title: Configure a cups-pdf printer on Cups and use it on AIX
wordpress_id: 141
categories:
- aix
- fast-tip
- linux/unix
- sysadmin
- trick
tags:
- aix
- cups
- cups-pdf
- linux
- pdf
- printers
- sysadmin
- tip
- trick
---

I will briefly describe how to set a cups-pdf on cups on Linux and configure AIX to use it. It is an easy task.



	
  1. Install on Linux cups and cups-pdf (for  SucksE (Suse) you can  find it in openSuse repositories).

The cups-pdf package configures automaticly a printer called "cups-pdf"

	
  2. You can access the CUPS configuration page via [http://localhost:631](http://localhost:631/).  If it is in a remote server, you can forward the port via SSH: "ssh -R  6310:localhost:631 host" and access via [http://localhost:631](http://localhost:6310/).

	
  3. To use it on AIX, you need to configure the LPD  protocol enabling cups-lpd in xinetd: On suse you must enable it in /etc/xinetd.d/cups-lpd.NOTE: You must disable the usage of banners (added by default by  cups-lpd when converting from lpd to ipp) or you will get always a file  called "Test_Page.pdf" with only the banner. I think that newer versions  of cups solve this problem. To do that, you must add to cups-lpd the  option -o job-sheets=none



    
    sed 's/\(disable.*=\).*/\1 no/' -i /etc/xinetd.d/cups-lpd	
    grep -q job-sheets=none /etc/xinetd.d/cups-lpd || sed 's/\(server_args.*=.*\)/\1 -o job-sheets=none/' -i /etc/xinetd.d/cups-lpd
    /etc/init.d/xinetd reload
    
    


Finally on AIX, you can create you new printer as a  BSD printer:

    
    /usr/lib/lpd/pio/etc/piomisc_ext mkpq_remote_ext  -q 'cups-pdf' -h 'remoteserver' -r 'cups-pdf' -t 'bsd' -C 'FALSE' -d 'Virtual PDF printer on remoteserver'
    
    


That is all. You can use your virtual pdf printer on  AIX: ls | lp -d cups-pdf

You may want tune some cups-pdf settings in /etc/cups/cups-pdf.conf,  like:



	
  * UserUMask 0007: This option affects the "umask" default ACL  configuration. If you set 0077 it will set umask=--- in final PDF, I do  not known why :?

    
    ### Key: UserUMask
    ##  umask for user output of known users
    ##  changing this can introduce security leaks if confidential
    ##  information is processed!
    ### Default: 0077
    
    UserUMask 0007
    




	
  * Label 1, to avoid overwrites...

    
    ### Key: Label
    ##  label all jobs with a unique job-id in order to avoid overwriting old
    ##  files in case new ones with identical names are created; always true for
    ##  untitled documents
    ##  0: label untitled documents only, 1: label all documents
    ### Default: 0
    






	
  * Paths, etc....


