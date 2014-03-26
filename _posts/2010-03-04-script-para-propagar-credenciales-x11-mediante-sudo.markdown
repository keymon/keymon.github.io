---
author: keymon
comments: true
date: 2010-03-04 12:47:21+00:00
layout: post
slug: script-para-propagar-credenciales-x11-mediante-sudo
title: Script para propagar credenciales X11 mediante sudo
wordpress_id: 30
categories:
- fast-tip
- script
- sysadmin
tags:
- bash
- propagate
- script
- ssh
- sudo
- unix
- xauth
---



Los usuarios-administradores de servicios que corren en mi máquina muchas veces tienen que ejecutar programas gráficos. Por supuesto, les tengo prohibido entrar con usuarios de sistema o servicios, y les obligo a usar sudo para convertirse en el usuario de turno.

Para simplificarles la vida, he hecho este script que propaga las credenciales X11, a un usuario destino usando sudo.

De esta forma podemos entrar en una máquina con nuestro usuario corporativo con las X11 activas (p.ej. con Xming) y poder usar el sistema gráfico con un usuario de sistema, como oracle o was.

En inglés: Script to propagate xauth (x11) credentials to other user using sudo.

El script:



	
  * Codigo fuente: [http://github.com/keymon/snippets/blob/master/scripts/misc/xauth-propagate-to-user.sh](http://github.com/keymon/snippets/blob/master/scripts/misc/xauth-propagate-to-user.sh)

	
  * Descarga: [http://github.com/keymon/snippets/raw/master/scripts/misc/xauth-propagate-to-user.sh](http://github.com/keymon/snippets/blob/master/scripts/misc/xauth-propagate-to-user.sh)


Para eso sólo hay que ejecutar:

    
    ./xauth-propagate-to-user.sh <usuario>
    


Por ejemplo:

    
    $ ./xauth-propagate-to-user.sh pepito
    Testing the display 'localhost:17.0'. Close the graphical program 'xclock' when displayed
    (...)
    Exporting dcdcorejees1/unix:12 to user pepito.
    myuser's password for sudo: *****
    WARNING, sudo does not propagate the $DISPLAY variable. It must be set manually:
     'sudo -H -u pepito DISPLAY=localhost:17.0 <command>'
    Testing the display 'localhost:17.0' for user 'pepito'. Close the graphical program 'xclock' when displayed
    (...)
    It works!!!
    To execute any command as 'pepito', use this command line:
     'sudo -H -u pepito DISPLAY=localhost:17.0 <command> [<args>]'
    
    So. to work with a shell with 'pepito', execute this command:'
     'sudo -H -u pepito DISPLAY=localhost:17.0 sh'
    


Luego, durante esa sesión, podremos convertirnos al usuario de turno y ejecutar comandos que requieran entorno gráfico.

IMPORTANTE: en el sudo debe indicarse la opción –H, para que establezca el $HOME al del usuario destino.


