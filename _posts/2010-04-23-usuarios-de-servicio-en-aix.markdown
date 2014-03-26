---
author: keymon
comments: true
date: 2010-04-23 07:53:40+00:00
layout: post
slug: usuarios-de-servicio-en-aix
title: Usuarios de servicio en AIX.
wordpress_id: 62
categories:
- aix
- fast-tip
tags:
- aix
- linux
- security
- sysadmin
- tip
- users
---



Cuando aparezca este error:

    
    Apr 23 08:43:41 myhost auth|security:info sshd[737348]: Login restricted for randomuser: There have been too many unsuccessful login attempts; please see \tthe system administrator.Apr 23 08:43:41 myhost auth|security:info sshd[737348]: Failed none for invalid user randomuser from 1.2.3.4 port 21693 ssh2


Se debe a que el número de fallos de autenticación ha sido superado. Se  pueden consultar el máximo (loginretries) y el número  (unsuccessful_login_count):

    
    lsuser -a loginretries -a unsuccessful_login_count randomuser


Para cambiarlo:

    
    chuser unsuccessful_login_count=0 randomuser


O actualizamos el fichero donde se guarda el contador con:

    
    chsec -f /etc/security/lastlog -a "unsuccessful_login_count=0" -s randomuser


Si el usuario es de sistema y no debe ser bloqueado, optamos por  cambiarle estos atributos, para que no se bloquee nunca ("man chuser"  para más info):

    
    admin=true # The user is an administrator. Only the root user can change the attributes of users defined as administrators.
    expires=0 # expiration date of the account. =0, the account does not
    expiremaxage=0 # the maximum age of a password. =0, no maximum age.
    maxexpired=-1 # maximum time a user can change an expired password.
    minage=0 # the minimum age of a password. =0, no minimum age.
    loginretries=-1 # Defines the number of unsuccessful login attempts allowed
    
    chuser admin=true expires=0 maxage=0 maxexpired=-1 minage=0 loginretries=-1 randomuser



