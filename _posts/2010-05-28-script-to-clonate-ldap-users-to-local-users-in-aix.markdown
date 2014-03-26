---
author: keymon
comments: true
date: 2010-05-28 12:34:20+00:00
layout: post
slug: script-to-clonate-ldap-users-to-local-users-in-aix
title: Script to clonate ldap users to local users in AIX
wordpress_id: 85
categories:
- aix
- fast-tip
- script
- sysadmin
tags:
- active
- activedirectory
- AD
- aix
- bash
- DA
- directory
- integracion
- secldapclntd
- sysadmin
- tip
- trick
---

AIX LDAP integration is not up to expectations. Its cache daemon, secldapclntd,
has a lot of problems:it often crashes, queries are slow, etc...

To mitigate problems, one workaround could be create the most important users locally,
using the KRB5files repository.

With this idea, this script will query a set of given groups from the AIX LDAP
registry using the AIX command line tools (lsuser, lsgroup), and it will create
them locally (mkgroup, mkuser).

To make it work, the host must be integrated with remote repository and must be able to resolve users and groups with LDAP method. You need LDAP method and KRB5files method configured. It can be easily changed to use other methods.

This script also supports nested groups from Active Directory.



	
  * Source: [http://github.com/keymon/snippets/blob/master/scripts/aix/clone-aix-ldap-groups.sh](http://github.com/keymon/snippets/blob/master/scripts/aix/clone-aix-ldap-groups.sh)

	
  * Download: [http://github.com/keymon/snippets/raw/master/scripts/aix/clone-aix-ldap-groups.sh](http://github.com/keymon/snippets/raw/master/scripts/aix/clone-aix-ldap-groups.sh)


