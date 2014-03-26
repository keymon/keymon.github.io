---
author: keymon
comments: true
date: 2010-09-17 09:53:21+00:00
layout: post
slug: el-run-parts-de-debian-y-etccron-d-no-soportan
title: El run-parts de debian y /etc/cron.d no soportan '.'
wordpress_id: 211
categories:
- fast-tip
- linux/unix
- Misc
- sysadmin
---

Hoy he perdido algo de tiempo con una soberana tonteria: Creé un script para ejecutar en el /etc/cron.daily llamado "script.sh"... y nunca se ejecutaba. Revisé todo arriba y abajo, y finalmente ejecuté:

    
    run-parts --report -v  --test /etc/cron.daily


y no salí el script... ¿que era? al final se me ocurrió quitarle el '.sh' del final... y voilá.

Y la cosa no acabó ahí. Otra definición en /etc/cron.d/zabbix.cron no cargaba... el mismo problema, sobraba el '.'.

En fins...
