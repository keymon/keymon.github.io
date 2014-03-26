---
author: keymon
comments: true
date: 2010-09-14 11:53:53+00:00
layout: post
slug: linux-nss-libnss-and-nss_ldap-a-working-configuration
title: Linux NSS (libnss) and nss_ldap a working configuration
wordpress_id: 207
categories:
- linux/unix
- sysadmin
- Technical
tags:
- activedirectory
- AD
- configuration
- ldap
- linux
- NSS
- nss_ldap
- sysadmin
---



In a previous post I commented the problems related to LDAP integration  of Linux with LDAP.  I proposed several solutions and commented that a  good configuration can be enough.  Tuning the configuration, trying to  avoid buggy code, minimizing locks and delays, etc...

In this post I will comment a configuration that is quite stable at the  moment, using Suse 11 SP2 with Active Directory 2003 + SFU.

<!-- more -->

As commented in previous posts, the nss_ldap+nscd architecture implies  lots of **concurrent** queries to the LDAP server and  constant retries when something fail (like LDAP server that is down).  So, IMHO, the best is avoid delays as much as possible.

You should have the last version of the software. At his moment we are  using:



	
  * nss_ldap-265-4.2

	
  * nscd-2.11.1-0.17.4


And we tuned the configuration, trying:

	
  * Reduce timeouts as much as possible.

	
  * Close the conections ASAP.

	
  * Avoid reconnections: The nscd+nss_ldap architecture makes it  reconect all the time.

	
  * If you have a big LDAP tree, try to configure the subtrees were  NIS data is stored, not the root DN.

	
  * In nscd, we following the recomendations in [http://anusf.anu.edu.au/~djh900/nscd.html](http://anusf.anu.edu.au/%7Edjh900/nscd.html):

	
    * Persistence is set to off, to avoid corruption.

	
    * Shared is on, to increase performance.





In this configuration I also set a special mapping:

	
  * cn: uid and cn

	
  * uniqueMember: member (Windows groups in Linux)

	
  * msSFU30MemberUid: msSFU30MemberUid


Other recomendations:

	
  * Use a different user for each host in rootbinddn.

	
  * Set ldap.secret and ldap.conf read only for root. So you avoid  other user processes to load ldap libraries.  Actually, the ideal would be run the nscd with a dedicated user, but I  was not able to make it work with nss_ldap :?


And here comes the configuration files:

nss_ldap.conf:

    
    # /etc/ldap.conf
    # Config file for nss_ldap.
    # This configuration tries to minimize delays due failures.
    
    # Logging
    # Specifies the directory used for logging by the LDAP client library. 
    # This feature is not supported by all client libraries.
    logdir /var/log/libnss-ldap.log
    
    # Specifies the debug level used for logging by the LDAP client library. 
    # This feature is not supported by all client libraries, and does not 
    # apply to the nss_ldap and pam_ldap modules themselves (debugging, if 
    # any, is configured separately and usually at compile time). 
    debug 0
    
    # =============================================================================
    # Connection configuration
    
    # Remote LDAP hosts:
    host 192.168.1.10 192.168.1.11 192.168.1.12 192.168.1.13
    
    # The port.
    port 389
    
    # LDAP Base DN
    base DC=mycompany,DC=org
    
    # The LDAP version to use (defaults to 3 if supported by client library)
    ldap_version 3
    
    # The distinguished name to bind to the server
    # Password is stored in /etc/ldap.secret (mode=600)
    rootbinddn CN=myhostldap,OU=hosts,DC=mycompany,DC=org
    
    # The search scope: one, base, sub
    scope sub
    
    # =============================================================================
    # Search timelimit.
    # In a local network we can set this to a low value. ~5s is good enough.
    timelimit 5
    
    # Bind/connect timelimit
    # In a local network we can set this to a low value. 
    bind_timelimit 2
    
    # Reconnect policy:
    #  hard_open: reconnect to DSA with exponential backoff if
    #             opening connection failed
    #  hard_init: reconnect to DSA with exponential backoff if
    #             initializing connection failed
    #  hard:      alias for hard_open
    #  soft:      return immediately on server failure
    bind_policy soft
    
    # Connection policy:
    #  persist:   DSA connections are kept open (default)
    #  oneshot:   DSA connections destroyed after request
    #nss_connect_policy oneshot
    
    # Idle timelimit; client will close connections
    # (nss_ldap only) if the server has not been contacted
    # for the number of seconds specified below.
    # With Active Directory I recomend <= 30.
    idle_timelimit 30
    
    # Reconnect policy.
    # Avoid delay in network outages by setting the lowest timeout and retries.
    # These are undocummented options!
    nss_reconnect_tries 1
    nss_reconnect_sleeptime 0
    nss_reconnect_maxsleeptime 0
    nss_reconnect_maxconntries 1
    
    # Use paged rseults
    nss_paged_results yes
    
    # Pagesize: when paged results enable, used to set the pagesize to a custom value.
    # We set it low. Bigger values caused me some trouble with Active Directory
    pagesize 100
    
    # Do not follow deferences. More random problems with this.
    deref never
    referrals no
    
    
    # =============================================================================
    # Enable support for RFC2307bis (distinguished names in group members)
    nss_schema      rfc2307bis
    
    # Use backlinks for answering initgroups()
    # This option directs the nss_ldap implementation of initgroups(3)
    # to determine a users group membership by reading  the  memberOf
    # attribute  of  their directory entry (and of any nested groups),
    # rather than querying on uniqueMember.
    #
    # Enabling this increases the performance
    nss_initgroups backlink
    
    # returns NOTFOUND if nss_ldap's initgroups() is called
    # for users specified in nss_initgroups_ignoreusers
    # (comma separated)
    nss_initgroups_ignoreusers      root
    
    # Specifies  whether  or  not  to populate the members list in the
    # group structure for group lookups. If  very  large  groups  are
    # present,  enabling this option will greatly increase perforance,
    # at the cost of some lost functionality.  You  should  verify  no
    # local applications rely on this information before enabling this
    # on a production system.
    #
    # Enable this increases the performance
    nss_getgrent_skipmembers yes
    
    # =============================================================================
    # Active Directory Mappings
    
    # Root of the company. Try to avoid add this, is better add only lower subtrees.
    #nss_base_passwd   DC=mycompany,DC=org
    #nss_base_group    DC=mycompany,DC=org
     
    nss_base_passwd   OU=users1,DC=mycompany,DC=org
    nss_base_group    OU=groups1,DC=mycompany,DC=org
    nss_base_passwd   OU=users2,DC=mycompany,DC=org
    nss_base_group    OU=groups2,DC=mycompany,DC=org
    nss_base_passwd   OU=staff,DN=subcompany,DC=mycompany,DC=org
    nss_base_group    OU=staff,DN=subcompany,DC=mycompany,DC=org
    
    
    # =============================================================================
    # Mapeo de Clases y attributos (Attribute/objectclass mapping).
    # Baseado en SFU con modificaciones
    
    # Clases de objectos
    nss_map_objectclass posixAccount    user
    nss_map_objectclass shadowAccount   user
    nss_map_objectclass posixGroup      group
    nss_map_objectclass ipHost          computer
    
    
    # Mapeo de atributos (comentado valor por defecto)
    nss_map_attribute cn                cn
    nss_map_attribute uid               cn
    nss_map_attribute uidNumber         msSFU30UidNumber
    nss_map_attribute gidNumber         msSFU30GidNumber
    nss_map_attribute homeDirectory     msSFU30HomeDirectory
    nss_map_attribute loginShell        msSFU30LoginShell
    nss_map_attribute gecos             displayName
    nss_map_attribute shadowLastChange  pwdLastSet
    nss_map_attribute uniqueMember      member
    nss_map_attribute memberUid         msSFU30MemberUid
    
    nss_map_attribute ipHostNumber      msSFU30IpHostNumber
    
    # =============================================================================
    # OpenLDAP SSL mechanism
    #ssl            no
    
    # OpenLDAP SSL options
    # Require and verify server certificate (yes/no)
    #tls_checkpeer no
    
    # CA certificates for server certificate verification
    # At least one of these are required if tls_checkpeer is "yes"
    #tls_cacertfile /etc/ssl/ca.cert
    #tls_cacertdir /etc/ssl/certs
    
    # Seed the PRNG if /dev/urandom is not provided
    #tls_randfile /var/run/egd-pool
    
    # SSL cipher suite
    # See man ciphers for syntax
    #tls_ciphers TLSv1
    
    # Client certificate and key
    # Use these, if your server requires client authentication.
    #tls_cert
    #tls_key
    
    # Disable SASL security layers. This is needed for AD.
    #sasl_secprops maxssf=0
    
    # Override the default Kerberos ticket cache location.
    #krb5_ccname FILE:/etc/.ldapcache
    


nscd.conf

    
    #
    # /etc/nscd.conf
    #
    # An example Name Service Cache config file.  This file is needed by nscd.
    #
    # Legal entries are:
    #
    #       logfile                 <file>
    #       debug-level             <level>
    #       threads                 <initial #threads to use>
    #       max-threads             <maximum #threads to use>
    #       server-user             <user to run server as instead of root>
    #               server-user is ignored if nscd is started with -S parameters
    #       stat-user               <user who is allowed to request statistics>
    #       reload-count            unlimited|<number>
    #       paranoia                <yes|no>
    #       restart-interval        <time in seconds>
    #
    #       enable-cache            <service> <yes|no>
    #       positive-time-to-live   <service> <time in seconds>
    #       negative-time-to-live   <service> <time in seconds>
    #       suggested-size          <service> <prime number>
    #       check-files             <service> <yes|no>
    #       persistent              <service> <yes|no>
    #       shared                  <service> <yes|no>
    #       max-db-size             <service> <number bytes>
    #       auto-propagate          <service> <yes|no>
    #
    # Currently supported cache names (services): passwd, group, hosts, services
    #
    
    #=====================================================================
    # Global settings
    
    # Debug settings
    logfile                 /var/log/nscd.log
    debug-level             0
    
    # Allow it to have plenty of threads
    threads                 4
    max-threads             32
    
    # Set different credentials for daemon. 
    #server-user             nobody
    #stat-user               nobody
    
    # Restart the daemon periodically.
    # Is goot set this option
    paranoia                yes
    restart-interval        10800
    
    # Sets the number of times a cached record is reloaded before it
    # is pruned from the cache.
    #reload-count            5
    
    #=====================================================================
    # Passwd settings
    
    enable-cache            passwd          yes
    positive-time-to-live   passwd          600
    negative-time-to-live   passwd          10
    suggested-size          passwd          211
    check-files             passwd          yes
    # Persistent = no, Shared = yes
    persistent              passwd          no
    shared                  passwd          yes
    max-db-size             passwd          33554432
    auto-propagate          passwd          yes
    
    # Group settings
    enable-cache            group           yes
    positive-time-to-live   group           600
    negative-time-to-live   group           20
    suggested-size          group           211
    check-files             group           yes
    # Persistent = no, Shared = yes
    persistent              group           no
    shared                  group           yes
    max-db-size             group           33554432
    auto-propagate          group           yes
    
    # Hosts settings
    enable-cache            hosts           yes
    positive-time-to-live   hosts           600
    negative-time-to-live   hosts           0
    suggested-size          hosts           211
    check-files             hosts           yes
    # Persistent = no, Shared = yes
    persistent              hosts           no
    shared                  hosts           yes
    max-db-size             hosts           33554432
    
    # services settings
    enable-cache            services        yes
    positive-time-to-live   services        28800
    negative-time-to-live   services        20
    suggested-size          services        211
    check-files             services        yes
    # Persistent = no, Shared = yes
    persistent              services        no
    shared                  services        yes
    max-db-size             services        33554432
    
    



