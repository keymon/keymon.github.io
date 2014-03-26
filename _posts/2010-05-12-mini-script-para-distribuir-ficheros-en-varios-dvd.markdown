---
author: keymon
comments: true
date: 2010-05-12 10:48:04+00:00
layout: post
slug: mini-script-para-distribuir-ficheros-en-varios-dvd
title: Mini-Script para distribuir ficheros en varios DVD
wordpress_id: 72
categories:
- coding
- fast-tip
- script
---

[sourcecode language="bash"]
# Dividir ficheros en directorios para gravar DVDs

SIZE=0
COUNT=0
for i in *.gz; do
 FSIZE=$(ls -l $i | awk '{print $5}')
 SIZE=$(($SIZE+$FSIZE/1024))

 echo "Moviendo $i ($FSIZE) a dvd$COUNT/ ($SIZE)"
 if [ "$SIZE" -gt "$((4500*1024))" ]; then
 COUNT=$(($COUNT+1))
 SIZE=0
 fi
 [ -d  dvd$COUNT ] || mkdir -p dvd$COUNT
 mv $i dvd$COUNT/
done
[/sourcecode]
