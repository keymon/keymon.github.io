---
author: keymon
comments: true
date: 2010-06-22 11:38:09+00:00
layout: post
slug: problem-setting-manifest-in-multiple-environments-in-puppet
title: Problem setting manifest in multiple environments in puppet.
wordpress_id: 137
categories:
- fast-tip
- puppet
- Technical
tags:
- configuration
- environments
- manifests
- puppet
- sysadmin
---

From [http://projects.reductivelabs.com/issues/1615](http://projects.reductivelabs.com/issues/1615)

I think that I had the same issue. I want to use multiple environments, but I had a problem, puppet was always looking for the default location /etc/puppet/conf/manifests/site.pp instead of /etc/puppet/conf/data/<env>/manifests/site.pp.

<!-- more -->

My configuration file looked like this:

    
    [test]
     manifestdir =     /etc/puppet/data/test/manifests
     
    [preproduction]
     manifestdir =     /etc/puppet/data/preproduction/manifests
     
    [development]
     manifestdir =     /etc/puppet/data/development/manifests
     
    [production]
     manifestdir =     /etc/puppet/data/production/manifests
     
    [puppetmasterd]
     environments =     test,development,preproduction,production
     manifest =     $manifestdir/site.pp
    


Well, the problem was the 'manifest' entry in puppetmasterd. Puppet will evaluate it globally before evaluate the environments, setting manifest=/etc/puppet/conf/manifests/site.pp for all the environments.

Solution:

You can think that simply comment the line "manifest = $manifestdir/site.pp", woudl work since it is the default, but not. Neither writing "manifest = $manifestdir/site.pp" in each environmnet stanza works.** You must set the full qualified path to the site.pp: manifest = /etc/puppet/conf/data/<env>/manifests/site.pp**

Indeed, I think that you can not use vars in the environment stanzas, but I have to confirm it.

You can test the values using these commands in the puppetmaster host:

    
    $ puppetd --environment development --configprint manifest --config /etc/puppet/puppet.conf
    /etc/puppet/conf/manifests/site.pp
    $ puppetd --environment development --configprint manifest --config /etc/puppet/puppet.conf
    /etc/puppet/conf/manifests/site.pp


And after set the manifest in each environment stanza:

    
    $ puppetd --environment development --configprint manifest --config /etc/puppet/puppet.conf
    /etc/puppet/data/development/site.pp


I read somewhere that manifestdir wil be obsolete in future versions.


Well, the problem was the 'manifest' entry in puppetmasterd. Puppet will evaluate it globally before evaluate the environments, setting manifest=/etc/puppet/conf/manifests/site.pp for all the environments.

Solution: simply comment the line "manifest = $manifestdir/site.pp". That is the default, so it is ok to comment it. Other option is to include it in all environment stanzas, so you will always overwrite it value.

I read somewhere that manifestdir wil be obsolete in future versions.


