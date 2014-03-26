---
author: keymon
comments: true
date: 2010-06-08 07:26:53+00:00
layout: post
slug: entendiendo-como-funcionan-las-credenciales-de-x11
title: Entendiendo como funcionan las credenciales de X11
wordpress_id: 114
categories:
- sysadmin
- Technical
- trick
tags:
- credentials
- sysadmin
- windows
- x11
- xauth
---

Sobre el post "[Script  para propagar credenciales X11 mediante sudo"](../2010/03/04/script-para-propagar-credenciales-x11-mediante-sudo/)...

Recomiendo que se entienda cómo funciona esto de las X. Os ayudará, y no sufriréis cuando las cosas no funcionen:



	
  1. Cuando te logueas, se crea:



	
  * La variable $DISPLAY=localhost:11.0. Eso indica que hay un servidor de X escuchando en el puerto TCP 6011 de localhost. En el caso del SSH, se crea un túnel virtual al servidor de X de tu PC.

	
  * Se da de alta una clave (como las claves de SSH o certificados de SSL) en .Xauthority. Se puede consultar con xauth:



    
    $ xauth list
    hostname/unix:14  MIT-MAGIC-COOKIE-1  6a45c2933b65cd936f3e9031c8553d75





	
  1. Cuando ejecutas el script, lo único que se hace es EXPORTAR la clave (xauth extract fichero <host>/11.0) e importárselo al usuario destino (xauth merge fichero), usando sudo para ejecutar los comandos.


3.  Al hacer sudo al usuario destino, debe permanecer la variable $DISPLAY. En ocasiones sudo está configurado para no propagar esa variable, por lo que si no tiene valor, se le debe asignar (de ahí lo de sudo -u usuario DISPLAY=... –s)

Esto puede fallar:

	
  1. Porque no se crea el $DISPLAY o la clave inicial. Esto a su vez es:

	
    1. Porque no está instalado el xauth o no está en el $PATH del usuario.

	
    2. Porque el servidor de SSH no tiene habilitado el soporte de _X forwarding_.

	
    3. Porque el $HOME del usuario no está accesible para escritura/lectura/bloqueo.

	
    4. Porque el servidor no tienes arrancado el servidor de X o bien configurado el _X forwarding: (_[http://www.cse.unsw.edu.au/~helpdesk/documentation/Putty.html](http://www.cse.unsw.edu.au/~helpdesk/documentation/Putty.html))






	
  1. Porque no se propagan las credenciales al usuario destino:

	
    1. Porque no está instalado el xauth o no está en el $PATH del usuario.

	
    2. Porque el $HOME del usuario destino no está accesible para escritura/lectura/bloqueo.

	
    3. Porque no se tiene permisos para hacer sudo.





