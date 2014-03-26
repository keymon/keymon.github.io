---
author: keymon
comments: true
date: 2010-12-27 12:21:24+00:00
layout: post
slug: how-to-use-github-behind-a-proxy-using-ssh-transparently
title: How to use github behind a proxy using ssh transparently
wordpress_id: 248
categories:
- fast-tip
- github
- linux/unix
- Technical
- trick
tags:
- git
- github
- linux
- ssh
---

If you are behind a proxy that allows HTTPS connections, you can use  github via SSH without problems. To do so, you have to use the great tool connect.c ([ http://bent.latency.net/bent/git/goto-san-connect-1.85/src/connect.html](http://bent.latency.net/bent/git/goto-san-connect-1.85/src/connect.html)).  As described in its homepage, this program tunnels a connection using a  proxy, to allow SSH to connect to servers using a proxy.

You can configure connect as the ProxyCommand for ssh.github.com and github.com hosts  in ~/.ssh/config. You can also set the Port to 443 aswell.

Basicly the process will be:

    
    export PROXY=proxy:80
    
    http_proxy=http://$PROXY wget http://www.taiyo.co.jp/~gotoh/ssh/connect.c -O /tmp/connect.c
    gcc /tmp/connect.c -o ~/bin/connect 
    
    cat >> ~/.ssh/config  <<EOF
    
    Host ssh.github.com github.com
      Port 443
      HostName ssh.github.com
      IdentityFile $HOME/.ssh/id_rsa
      ProxyCommand $HOME/bin/connect -H proxy:80 %h %p
    
    EOF


And ready!!

    
    git clone git@github.com:keymon/facter.git facter


Easy, isn't it?

Check connect.c documentation if you need to use an authenticated user  in proxy.
