---
author: keymon
comments: true
date: 2010-03-31 12:21:47+00:00
excerpt: How to compile nss_ldap on AIX 6.1
layout: post
slug: compiling-nss_ldap-on-aix-6-1
title: Compiling nss_ldap on AIX 6.1
wordpress_id: 43
---




## Prereqs


We are using AIX 6.1 TL4 SP2:

    
    $ oslevel -s
    6100-04-02-1007
    


Installed openldap library from AIX-[ToolboxForLinuxApplications?](http://trac.dclinuxapps1.caixagalicia.cg/wiki/ToolboxForLinuxApplications) Cd (we are using  06.2007)

    
    $ rpm -qa |grep ldap
    openldap-2.0.21-5ssl
    openldap-devel-2.0.21-5ssl
    




## Getting source


From [http://www.padl.com/OSS/nss_ldap.html](http://www.padl.com/OSS/nss_ldap.html)

Actually we are using nss_ldap-265

    
    wget http://www.padl.com/download/nss_ldap.tgz
    tar -xvzf nss_ldap.tgz
    cd nss_ldap-265
    




## Compiling




### Small fixes


First change the Makefile.am:

    
    cp  Makefile.am  Makefile.am.orig
    sed 's/    CVSVERSIONDIR=$(top_srcdir) vers_string -v/    CVSVERSIONDIR=$(top_srcdir)\/vers_string -v/' < Makefile.am.orig  > Makefile.am
    automake
    


Fix an small bug:

    
    $ diff aix_authmeth.c aix_authmeth.c~
    551c551
    <   int stat = _nss_ldap_parse_int(vals[0], 0, &av->attr_un.au_int);
    ---
    >   stat = _nss_ldap_parse_int(vals[0], 0, &av->attr_un.au_int);
    
    




### configure


We are going to use this command:

    
    LIBPATH=/usr/lib \
    ./configure \
        --with-ldap-dir=/opt/IBM/ldap/V6.1/ \
        --with-ldap-lib=auto\
        --with-ldap-conf-file=/etc/ldap.conf \
        --enable-rfc2307bis \
        --prefix=/usr/local/stow/nss_ldap-265
    
    


We add the variable LIBPATH=/usr/lib  because if not we get this error, I  do no known why:

    
    $./configure
    ...
    checking for struct ether_addr... yes
    checking for socklen_t... yes
    checking for pw_change in struct passwd... no
    checking for pw_expire in struct passwd... no
    checking for unsigned int... yes
    checking size of unsigned int... configure: error: cannot compute sizeof (unsigned int), 77
    See `config.log' for more details.
    
    And in config.log:
    ...
    configure:11621: ./conftest
    Could not load program ./conftest:
            Dependent module libnsl.a(shr.o) could not be loaded.
    Could not load module libnsl.a(shr.o).
    System error: No such file or directory
    configure:11624: $? = 255
    configure: program exited with status 255
    ...
    




### Compile



    
    make
    




### Install



    
    $ sudo make install
    make[1]: Entering directory `/mnt/cgx001/SoftwareRepository/source/nss_ldap-265'
    /bin/sh ./mkinstalldirs /usr/local/stow/nss_ldap-265/lib/netsvc/dynload
    ./install-sh -c -o root -g system nss_ldap.so /usr/local/stow/nss_ldap-265/lib/netsvc/dynload/nss_ldap.so
    /bin/sh ./mkinstalldirs /usr/local/stow/nss_ldap-265/lib/security
    mkdir /usr/local/stow/nss_ldap-265/lib/security
    ./install-sh -c -o root -g system NSS_LDAP /usr/local/stow/nss_ldap-265/lib/security/NSS_LDAP
    ./install-sh -c -m 644 -o root -g system ./nsswitch.ldap /usr/local/stow/nss_ldap-265/etc/nsswitch.ldap;
    test -z "/usr/local/stow/nss_ldap-265/man/man5" || /bin/sh ./mkinstalldirs "/usr/local/stow/nss_ldap-265/man/man5"
    mkdir /usr/local/stow/nss_ldap-265/man
    mkdir /usr/local/stow/nss_ldap-265/man/man5
     ./install-sh -c -m 644 './nss_ldap.5' '/usr/local/stow/nss_ldap-265/man/man5/nss_ldap.5'
    make[1]: Leaving directory `/mnt/cgx001/SoftwareRepository/source/nss_ldap-265'
    mkdir -p /usr/local/etc/ /usr/local/lib/netsvc /usr/local/lib/security /usr/local/man/man5
    cd /usr/local/stow
    stow nss_ldap-265/
    




### Configure



    
    cat >> /usr/lib/security/methods.cfg <<EOF 
    
    NSSLDAP:
            program = /usr/local/lib/security/NSS_LDAP
    
    KRB5NSSLDAP:
            options = db=NSSLDAP,auth=KRB5
    




## Testing


Rigth now is not operative.

It takes a lot of time to resolve things a name. With only 2 OUs and one  level search:

    
    $ time id ldapuser
    uid=....
    
    real    0m21.016s
    user    0m0.016s
    sys     0m0.022s
    


I have to debug that... I will update this case post.


