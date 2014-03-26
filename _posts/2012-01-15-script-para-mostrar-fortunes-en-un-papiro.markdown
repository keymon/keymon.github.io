---
author: keymon
comments: true
date: 2012-01-15 04:33:27+00:00
layout: post
slug: script-para-mostrar-fortunes-en-un-papiro
title: Script para mostrar fortunes en un papiro
wordpress_id: 109
categories:
- fast-tip
- Misc
- Personal
- script
tags:
- bash
- fortune
- motd
- script
---

Hace un porrón de años escribí esto... creo que aún estaba en el instituto :)

Básicamente funciona así (se ve mal por culpa de wordpress que quita los espacios... paso de rayarme):


    
    $ fortune | ./papel.sh
     __,-----------------------------------------------------------------------,__
    /  \\                                                                       \ \
    \__/|       Many a writer seems to think he is never profound except         |/
     __ |       when he can't understand his own meaning.                        |
    /  \|       -- George D. Prentice                                            |\
    \__//                                                                        //
       `------------------------------------------------------------------------``
    [sourcecode language="bash"]
    #!/bin/sh
    # Poner en el cron:
    # 33 *     * * *     root   (cat /etc/motd_base ; /usr/games/fortune | /usr/local/bin/papel.sh) > /etc/motd; chmod 644 /etc/motd
    
    fmt -w 60 | (
    read a;
    read b;
    read c;
    
    cat << "TOP"
     __,-----------------------------------------------------------------------,__
    /  \\                                                                       \ \
    TOP
    
    printf "\\__/|       %-65s|/\n" "$a"
    a=$b;
    b=$c;
    read c;
    
    while read c; do
    	printf "    |       %-65s|\n" "$a";
    	a=$b;
    	b=$c;
    done;
    
    printf " __ |       %-65s|\n" "$a";
    printf "/  \\|       %-65s|\\ \n" "$b";
    
    cat << "BOTTON"
    \__//                                                                        //
       `------------------------------------------------------------------------``
    BOTTON
    
    )
    [/sourcecode]
