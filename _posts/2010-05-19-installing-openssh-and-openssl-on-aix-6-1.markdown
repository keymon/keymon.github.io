---
author: keymon
comments: true
date: 2010-05-19 12:49:27+00:00
layout: post
slug: installing-openssh-and-openssl-on-aix-6-1
title: 'Installing OpenSSH and OpenSSL on AIX 6.1 '
wordpress_id: 82
categories:
- aix
- fast-tip
- sysadmin
- trick
tags:
- aix
- installation
- openssh
- openssl
- rpm
- rpm openssh openssl aix installation
- ssh
- ssl
- sysadmin
- tip
- trick
---

There are a lot of sites where the process is explained, just google a  little bit.

I will describe the one that worked for me:



	
  * I am using AIX 6.1 TL4 (upgraded from AIX 5.3 TL6)

	
  * I have the AIX Toolbox For Linux Applications of 05.2009


I prefer to have openssh and openssl as native AIX packages. The problem  is that .rpm files usually have dependencies with openssl, so I had to  install the openssl rpm package as well.

First I downloaded both openssl 0.9.8.1103 (from IBM) and openssh 5.2p1  (from Sourceforge):

	
  * [https://www14.software.ibm.com/webapp/iwm/web/reg/download.do?source=aixbp&S_PKG=openssl](https://www14.software.ibm.com/webapp/iwm/web/reg/download.do?source=aixbp&S_PKG=openssl)

	
  * [http://sourceforge.net/projects/openssh-aix/files/openssh-aix61/openssh_5.2p1_aix61.tar.Z/download](http://sourceforge.net/projects/openssh-aix/files/openssh-aix61/openssh_5.2p1_aix61.tar.Z/download)


I installed it:
`
mkdir openssh_5.2p1_aix61 && cd openssh_5.2p1_aix61 && uncompress -c < ../openssh_5.2p1_aix61.tar.Z |tar -xvf - && installp -acXYgd . openssh
mkdir openssl.0.9.8.1103 && cd openssl.0.9.8.1103 && uncompress -c < ../openssl.0.9.8.1103.tar.Z |tar -xvf - && installp -acXYgd . openssl
`

Then I downloaded a compiled version of openssl 0.9.8 from perzl.org  (AIX Toolbox For Linux Applications of 05.2009 comes with 0.9.7, but  .rpms have dependencies on 0.9.8):  [http://www.perzl.org/aix/index.php?n=Main.Openssl](http://www.perzl.org/aix/index.php?n=Main.Openssl).  And I installed it rpm -i openssl-0.9.8n-1.aix5.1.ppc.rpm.
