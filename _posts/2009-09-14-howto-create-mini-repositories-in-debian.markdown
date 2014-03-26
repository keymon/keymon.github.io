---
author: keymon
comments: true
date: 2009-09-14 10:08:28+00:00
layout: post
slug: howto-create-mini-repositories-in-debian
title: Howto create Mini-repositories in debian
wordpress_id: 19
categories:
- debian
- fast-tip
---






This tip allow us to create a small local repository to use with apt-get.

Package required: apt-utils.

If you have the packages in /any_path/repository, generate the Packages.gz with this commands:

    
    apt-ftparchive packages /any_path/repository > Packages
    gzip < /any_path/repository/Packages > /any_path/repository/Packages.gz


Or using this small script:

    
    #!/bin/sh
    cd $1
    apt-ftparchive packages . | tee Packages | gzip > Packages.gz


And you simply need to add in /etc/apt/sources.list:

    
    deb copy:/any_path/ repositorio


Simply...



