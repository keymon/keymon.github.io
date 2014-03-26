---
author: keymon
comments: true
date: 2010-03-09 16:55:35+00:00
excerpt: 'AIX has the following (inefficient) mechanism to resolve network names (DNS):
  It will ask each nameserver in /etc/resolv.conf always secuencially, one per one,
  waiting an amount of time (RES_TIMEOUT) until each of the servers respond or timeouts.
  If all of them are down, it will retry again defined number of times (RES_RETRY),
  doubling the timeout each cicle. This is really annoying specially in the worse
  moment: when you have problems.


  In this post I will test and comment this behaviour and propose some solutions.'
layout: post
slug: the-weird-domain-name-resolution-mechanism-on-aix
title: 'The weird domain name resolution mechanism on AIX '
wordpress_id: 35
categories:
- aix
- linux/unix
- sysadmin
tags:
- aix
- dns
- sysadmin
- tuning
---

AIX has the following (inefficient) mechanism to resolve network names (DNS):  It will ask each nameserver in /etc/resolv.conf always **secuencially**,  one per one, waiting an amount of time (RES_TIMEOUT) until each of the servers respond  or timeouts. If all of them are down, it will retry again defined number  of times (RES_RETRY), doubling the timeout each cicle. This is really annoying specially in the worse moment: when you have problems.

In this post I will test and comment this behaviour and propose some solutions.

<!-- more -->

All the test in this post were executed on a AIX 6.1 TL4 SP2.

In the following site you can see how control the timeout and the number of retries using the variables RES_TIMEOUT and RES_RETRY [http://publib.boulder.ibm.com/infocenter/aix/v6r1/index.jsp?topic=/com.ibm.aix.progcomm/doc/progcomc/skt_dns.htm](http://publib.boulder.ibm.com/infocenter/aix/v6r1/index.jsp?topic=/com.ibm.aix.progcomm/doc/progcomc/skt_dns.htm)



	
  * _RES_TIMEOUT: Overrides the default value of the retrans field of the _res structure, which is the value of the RES_TIMEOUT constant defined in the /usr/include/resolv.h file. This value is the base time-out period in seconds between queries to the name servers. After each failed attempt, the time-out period is doubled. The time-out period is divided by the number of name servers defined. The minimum time-out period is 1 second._

	
  * _RES_RETRY Overrides the default value for the retry field of the _res structure, which is 4. This value is the number of times the resolver tries to query the name servers before giving up. Setting RES_RETRY to 0 prevents the resolver from querying the name servers._


So, if your nameservers are not reachable you'll have a big problem: Most programs will fail with timeouts due name resolution,  you maybe can not enter in the hosts depending on your configuration. If the  hosts is starting it will last ages and some services will not start at all...

Let's do some tests with different values of these variables

    
    for t in {0..3}; do
        for r in {0..3}; do
            echo -n "RES_TIMEOUT=$t RES_RETRY=$r  "
            (RES_TIMEOUT=$t RES_RETRY=$r time ping -c 1 -w 1 xda) 2>&1 | grep -i Real
        done
    done
    


With first nameserver of four down:

    
    RES_TIMEOUT=0 RES_RETRY=0  Real   0.00
    RES_TIMEOUT=0 RES_RETRY=1  Real   1.00
    RES_TIMEOUT=0 RES_RETRY=2  Real   1.00
    RES_TIMEOUT=0 RES_RETRY=3  Real   1.00
    RES_TIMEOUT=1 RES_RETRY=0  Real   0.00
    RES_TIMEOUT=1 RES_RETRY=1  Real   1.00
    RES_TIMEOUT=1 RES_RETRY=2  Real   1.00
    RES_TIMEOUT=1 RES_RETRY=3  Real   1.00
    RES_TIMEOUT=2 RES_RETRY=0  Real   0.10
    RES_TIMEOUT=2 RES_RETRY=1  Real   2.01
    RES_TIMEOUT=2 RES_RETRY=2  Real   2.01
    RES_TIMEOUT=2 RES_RETRY=3  Real   2.00
    RES_TIMEOUT=3 RES_RETRY=0  Real   0.00
    RES_TIMEOUT=3 RES_RETRY=1  Real   3.00
    RES_TIMEOUT=3 RES_RETRY=2  Real   2.95
    RES_TIMEOUT=3 RES_RETRY=3  Real   3.00
    


With two firsts nameservers of four down:

    
    RES_TIMEOUT=0 RES_RETRY=0  Real   0.10
    RES_TIMEOUT=0 RES_RETRY=1  Real   2.00
    RES_TIMEOUT=0 RES_RETRY=2  Real   1.99
    RES_TIMEOUT=0 RES_RETRY=3  Real   2.00
    RES_TIMEOUT=1 RES_RETRY=0  Real   0.00
    RES_TIMEOUT=1 RES_RETRY=1  Real   2.00
    RES_TIMEOUT=1 RES_RETRY=2  Real   2.00
    RES_TIMEOUT=1 RES_RETRY=3  Real   2.00
    RES_TIMEOUT=2 RES_RETRY=0  Real   0.00
    RES_TIMEOUT=2 RES_RETRY=1  Real   4.00
    RES_TIMEOUT=2 RES_RETRY=2  Real   4.00
    RES_TIMEOUT=2 RES_RETRY=3  Real   4.10
    RES_TIMEOUT=3 RES_RETRY=0  Real   0.00
    RES_TIMEOUT=3 RES_RETRY=1  Real   5.99
    RES_TIMEOUT=3 RES_RETRY=2  Real   6.00
    RES_TIMEOUT=3 RES_RETRY=3  Real   6.04
    


With all four nameserver hosts down:

    
    RES_TIMEOUT=0 RES_RETRY=0  Real   0.00
    RES_TIMEOUT=0 RES_RETRY=1  Real   12.10
    RES_TIMEOUT=0 RES_RETRY=2  Real   24.00
    RES_TIMEOUT=0 RES_RETRY=3  Real   36.00
    RES_TIMEOUT=1 RES_RETRY=0  Real   0.00
    RES_TIMEOUT=1 RES_RETRY=1  Real   12.04
    RES_TIMEOUT=1 RES_RETRY=2  Real   20.00
    RES_TIMEOUT=1 RES_RETRY=3  Real   35.94
    RES_TIMEOUT=2 RES_RETRY=0  Real   0.00
    RES_TIMEOUT=2 RES_RETRY=1  Real   24.04
    RES_TIMEOUT=2 RES_RETRY=2  Real   40.04
    RES_TIMEOUT=2 RES_RETRY=3  Real   72.00
    RES_TIMEOUT=3 RES_RETRY=0  Real   0.00
    RES_TIMEOUT=3 RES_RETRY=1  Real   36.00
    RES_TIMEOUT=3 RES_RETRY=2  Real   60.00
    RES_TIMEOUT=3 RES_RETRY=3  Real   108.03
    


With only down nameserver host configured:

    
    RES_TIMEOUT=0 RES_RETRY=0  Real   0.00
    RES_TIMEOUT=0 RES_RETRY=1  Real   4.00
    RES_TIMEOUT=0 RES_RETRY=2  Real   8.00
    RES_TIMEOUT=0 RES_RETRY=3  Real   12.00
    RES_TIMEOUT=1 RES_RETRY=0  Real   0.00
    RES_TIMEOUT=1 RES_RETRY=1  Real   4.00
    RES_TIMEOUT=1 RES_RETRY=2  Real   12.10
    RES_TIMEOUT=1 RES_RETRY=3  Real   28.01
    RES_TIMEOUT=2 RES_RETRY=0  Real   0.00
    RES_TIMEOUT=2 RES_RETRY=1  Real   7.97
    RES_TIMEOUT=2 RES_RETRY=2  Real   24.00
    RES_TIMEOUT=2 RES_RETRY=3  Real   56.00
    RES_TIMEOUT=3 RES_RETRY=0  Real   0.00
    RES_TIMEOUT=3 RES_RETRY=1  Real   12.00
    RES_TIMEOUT=3 RES_RETRY=2  Real   36.04
    RES_TIMEOUT=3 RES_RETRY=3  Real   84.04
    


I also checked executing two queryies from the same process, an the result is the same:

    
    $ time RES_TIMEOUT=1 RES_RETRY=1 ./test.pl host01 host02
    14:13:01
    host01 resolves to 192.168.1.2
    14:13:12
    host02 resolves to 192.168.1.2
    14:13:22
    


From these tests we can say:



	
  * RES_TIMEOUT=0 is the same than RES_TIMEOUT=1, as expected.

	
  * RES_RETRY=0 always fails

	
  * It always query the hosts in the same order and it doesn't track failing nameservers (Even in the same process). So, if firsts nameserver  fails, you always has a delay in resolution.

	
  * the more nameservers you have the more delay you get if all of them fail. Documentation say that the RES_TIMEOUT is divided by the number of nameservers, but that is done BEFORE it multiplies the timeout by the loop counter.


The default values for this variables are RES_TIMEOUT=5 and RES_RETRY=4. With four nameservers failing that means:

    
    $ (RES_TIMEOUT=5 RES_RETRY=4 time ping -c 1 -w 1 xda) 2>&1 | grep -i real
    Real   339.91
    


THAT'S MORE THAN 5m!!! just to resolv one hostname!

What about GNU implementation? I did not test it too far, maybe another day, but my experience says that the GNU  implementation works better by default (and much better if they use nscd).  Inded, the GNU programs compiled in AIX using GCC (they are supposed to use glibc) fail faster too:

    
    $ time /opt/freeware/bin/wget http://host01
    --14:38:03--  http://host01
               => `index.html'
    Resolving host01... failed: Host not found.
    
    Real   6.00
    User   0.00
    System 0.00
    $ RES_TIMEOUT=1 RES_RETRY=1 time ping -c 1 -w 1 host01
    0821-062 ping: host name host01 NOT FOUND
    
    Real   11.98
    User   0.00
    System 0.00
    




### Proposed solutions


Firstly, IMHO, the best settings are the minimum value for both variables. It is better to have a application complaining about resolution problems than have it waiting endless (at least you would know what is failing).

    
    cat <<EOF >> /etc/environment 
    
    # AIX Name resolution settings
    RES_TIMEOUT=1
    RES_RETRY=1
    
    EOF
    


Secondly, try to organize the nameservers in /etc/resolv.conf in a failure order. For instance, if you have 2 sites and 2 DNS in each site, use this resolv.conf:

    
    nameserver      192.168.1.1 # DNS 1 in site 1
    nameserver      192.168.1.3 # DNS 1 in site 2
    nameserver      192.168.1.2 # DNS 2 in site 1
    nameserver      192.168.1.4 # DNS 2 in site 2
    domain  mydomain.com
    


More complex solutions involve some extra skills. For instance, you can also use an script, scheduled in cron or using mmonit ([http://mmonit.com/](http://mmonit.com/)),  that would check the nameserver availability and reorganize or comment out the  entries in /etc/resolv.conf.

You can also set a DNS server or DNS proxy system in the localhost. Right now I am testing this last solution, by using pdnsd: [http://www.phys.uu.nl/~rombouts/pdnsd.html](http://www.phys.uu.nl/%7Erombouts/pdnsd.html) . This software will allow us to tune in detail our DNS client in a Unix BOX.

Soon I will comment  my results in this blog.




This behaviour is weird:



	
  * It always query the hosts in the same order and it doesn't track failing nameservers (Even in the same process). So, if firsts nameserver  fails, you always has a delay in resolution.



	
  * the more nameservers you have the more delay you get if all of them fail. Documentation say that the RES_TIMEOUT is divided by the number of nameservers, but that is done BEFORE it multiplies the count of retries.



