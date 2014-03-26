---
author: keymon
comments: true
date: 2010-04-20 11:55:34+00:00
excerpt: "Voy a hablar de las conclusiones empíricas a las que hemos llegado implementando\
  \ la integración de Unix (En nuestro caso AIX y Linux) con el Directorio Activo.\
  \ Se han descubierto varios problemas, limitaciones, soluciones alternativas y recomendaciones\
  \ que intentaré resumir ahora.\n\nPara integrar el Directorio Activo con los sistemas\
  \ Unix lo mejor es emplear LDAP (consulta de usuarios, grupos y sus atributos y\
  \ relaciones) y Kerberos para autenticar.  Para soportar atributos Unix, se extendió\
  \ el esquema del DA con las extensiones SFU 3.5 (Services For Unix). Esto agrega\
  \ nuevos atributos, entre los que destacan:\n\n    * msSFU30Name             Nombre\
  \ del usuario en Unix. Puede ser substituido por ‘cn’ (ver abajo)\n    * msSFU30UidNumber\
  \        Identificador UNIX del usuario\n    * msSFU30GidNumber        Identificador\
  \ UNIX del grupo. Dentro de un usuario significa el grupo por defecto.\n    * msSFU30HomeDirectory\
  \         Directorio $HOME del usuario.\n    * msSFU30LoginShell Interprete de comandos\
  \ por defecto del usuario.\n    * msSFU30PosixMember     Referencia a miembros de\
  \ un grupo (DN del LDAP). Puede ser substituido por ‘member’ (ver abajo)\n    *\
  \ msSFU30MemberUid        Referencia a miembros de un grupo (Mediante UID del usuario).\
  \ Prácticamente no se usa.\n\nPara cambiar los atributos Unix en el Directorio Activo,\
  \ debe instalarse una .dll en el sistema del operador para agregar estas nuevas\
  \ pestañas al diálogo de propiedades de usuarios y grupos.\n\nVeamos los problemas\
  \ y limitaciones detectados y posibles soluciones"
layout: post
slug: integracion-de-sistemas-unix-aix-y-linux-con-directorio-activo-problemas-reales-y-posibles-soluciones
title: Integración de sistemas Unix (AIX y Linux) con Directorio Activo. Problemas
  reales y posibles soluciones.
wordpress_id: 52
categories:
- Active Directory
- aix
- linux/unix
- sysadmin
tags:
- active
- activedirectory
- AD
- aix
- DA
- directory
- integracion
- kerberos
- ldap
- linux
- nss_ldap
- resolucion
- resolver
- SFU
- sysadmin
- windows
---

Voy a hablar de las conclusiones empíricas a las que hemos llegado implementando la integración de Unix (En nuestro caso AIX y Linux) con el Directorio Activo. Se han descubierto varios problemas, limitaciones, soluciones alternativas y recomendaciones que intentaré resumir ahora.

Para integrar el Directorio Activo con los sistemas Unix lo mejor es emplear LDAP (consulta de usuarios, grupos y sus atributos y relaciones) y Kerberos para autenticar.  Para soportar atributos Unix, se extendió el esquema del DA con las extensiones SFU 3.5 (Services For Unix). Esto agrega nuevos atributos, entre los que destacan:



	
  * msSFU30Name             Nombre del usuario en Unix. Puede ser substituido por ‘cn’ (ver abajo)

	
  * msSFU30UidNumber        Identificador UNIX del usuario

	
  * msSFU30GidNumber        Identificador UNIX del grupo. Dentro de un usuario significa el grupo por defecto.

	
  * msSFU30HomeDirectory         Directorio $HOME del usuario.

	
  * msSFU30LoginShell Interprete de comandos por defecto del usuario.

	
  * msSFU30PosixMember     Referencia a miembros de un grupo (DN del LDAP). Puede ser substituido por ‘member’ (ver abajo)

	
  * msSFU30MemberUid        Referencia a miembros de un grupo (Mediante UID del usuario). Prácticamente no se usa.


Para cambiar los atributos Unix en el Directorio Activo, debe instalarse una .dll en el sistema del operador para agregar estas nuevas pestañas al diálogo de propiedades de usuarios y grupos.

Veamos los problemas y limitaciones detectados y posibles soluciones:

	
  * Problema: Cuando se cambia el nombre de usuario, el atributo Unix ‘msSFU30Name’ no cambia. Esto es bueno y malo, porque por un lado puede implicar incongruencias , pero por otro nos permite tener un nombre en Windows y otro en Unix más ‘_unix-friendly’_.Editarlo es un incordio, debe hacerse a mano con el ‘adsiedit’.

Solución: Usar el atributo ‘cn’ e vez de ‘msSFU30Name’



	
  * Problema: Los grupos del mundo Unix se gestionan separados de los del mundo Windows. La ventana de gestión de grupos Unix es tediosa e incómoda (te lista todos los usuarios por UID). No se pueden agregar grupos anidados.Solución: Usar el atributo ‘member’ en vez de ‘msSFU30PosixMember’. Se corresponde con los grupos Windows. Permite grupos anidados, que se aprovechan en Linux (nss_ldap).



	
  * AIX no soporta grupos anidados.
Soluciones:

	
    * No usar grupos anidados.



	
    * Clonar el DA “aplanando” los grupos en un servidor LDAP alternativo mediante un proceso de lotes.

	
    * Compilar y usar nss_ldap en AIX. No se recomienda: no está soportado y es extremadamente lento si hay muchos usuarios y grupos.






	
  * nss_ldap (Linux) soporta grupos anidados, pero NO sigue los grupos anidados para el _Primary Group_.
Solución: No usar grupos anidados para el _Primary Group_ o establecer un grupo _Primary Group_ de relleno.



	
  * Problema: El grupo por defecto de Windows (habitualmente _Domain Users_) no pueden leerlo ningún Unix. Para esta referencia se guarda el SID del grupo (última parte del GUID) en el atributo LDAP ‘primaryGroupID’.
Se suele recomendar que el grupo por defecto de Windows no se emplee para nada, pero muchas veces en las organizaciones suelen usarlo.

Solución: Podemos poner el mismo grupo en el _Primary Group_ de la pestaña Unix. Así tenemos total consistencia.

Problema de esta solución: En ocasiones necesitamos que el _Primary Group_ de Unix sea un grupo en particular, para que los ficheros creados pertenezcan a este usuario.



	
  * Problema: El no interpretar el grupo por defecto Windows, y no soportar grupos anidados en el _Primary Group_ (ver arriba), tendremos problemas si usamos grupos anidados con los grupos por defecto (como _Domain Users_).
Solución: Clonar los grupos por defecto de Windows. No usarlos para dar premisos.



	
  * Problema: Cuando hay muchos grupos con atributos Unix en el dominio, la pestaña de Primary Group no los muestra todos (posible bug).
Solución: asignar el Primary Group de forma numérica.



	
  * Problema: En AIX con el ‘secldapclntd’ y en Linux con ‘nscd’, que conecta al LDAP, pierden en rendimiento  y estabilidad si hay muchos usuarios/grupos y agregamos la rama raíz en la configuración.
Solución:

	
    * Intentar usar solo las ramas necesarias.

	
    * Lidiar con el soporte de IBM.

	
    * Tener la última versión actualizada.

	
    * Configurar monitores de servicio.

	
    * El rendimiento de nscd se puede aumentar configurando:


	
      * nss_initgroups backlink: Uso de “memberOf”

	
      * nss_getgrent_skipmembers yes: No generar listas de miembros de los grupos.










	
  * Problema: En AIX (si se usa el framework de autenticación soportado y por defecto: LAM) no se pueden mezclar usuarios y grupos locales y en LDAP. Si existe el mismo usuario en local y en LDAP, prevalece el local. Pero si un usuario LDAP pertenece a un grupo LDAP que también está definido en local, el funcionamiento es el esperado. En Linux sí se mezclan los grupos.



	
  * Problema: Si existe un mismo usuario o grupo con distintos UID o GID definidos en local y LDAP, el funcionamiento es aleatorio (válido para Linux y AIX).
Solución: Mantener consistencia entre la BD local y la remota.



	
  * Problema: Directorio Activo cierra conexiones a los 300 segundos. AIX o Linux pueden fallar la resolución aleatoriamente.
Solución: Bajar el tiempo de _heartbeat_ (o _idle time limit_ )en los clientes.



	
  * A tener en cuenta: Directorio Activo agrega un atributo LDAP a los usuarios, ‘memberOf’, que lista los grupos a los que pertenece el usuario o grupo para acelerar las consultas. nss_ldap  (Linux) permite usar este atributo si se emplea el atributo ‘member’ para los grupos.


Conclusiones y recomendaciones finales:

	
  * No usar los grupos por defecto de Windows para asignar permisos.



	
  * Definir el mismo _Primary Group_ que el grupo por defecto de Windows.



	
  * Evitar el uso de grupos anidados en AIX.



	
  * **Los usuarios y grupos en Unix en minúsculas y máximo 8 caracteres**. Nunca con caracteres como acentos, eñes…
AIX y Linux no soportan otro tipo explícitamente nombres más largos, y aunque “funcionan”, pueden dar problemas. Por ejemplo, las mayúsculas en el programa sudo se emplean para “Alias”.



	
  * Usar las relaciones grupos de Windows en vez de los de Unix (usar atributo ‘member’ en vez de ‘msSFU30PosixMember’



	
  * Usar este mapeo de atributos (ala nss_ldap):
nss_map_attribute cn                cn
nss_map_attribute uidNumber         msSFU30UidNumber
nss_map_attribute gidNumber         msSFU30GidNumber
nss_map_attribute homeDirectory     msSFU30HomeDirectory
nss_map_attribute loginShell        msSFU30LoginShell
nss_map_attribute gecos             displayName
nss_map_attribute shadowLastChange  pwdLastSet
nss_map_attribute uniqueMember      member
nss_map_attribute memberUid         msSFU30MemberUid
nss_map_attribute ipHostNumber      msSFU30IpHostNumber



	
  * Bajar el heartbeat o el timeout a 250s.

	
    * Nss_ldap de Linux, en ‘/etc/ldap.conf’: ‘idle_timelimit 250’

	
    * AIX, en ‘/etc/security/ldap/ldap.cfg’: heartbeatinterval: 250






	
  * El Directorio Activo no soporta _queries_ LDAP anónimas, debe usarse un usuario:

	
    * Es conveniente tener un usuario por host

	
    * **que este no se pueda bloquear**, si no se quiere abrir un potencial y simple ataque de _DoS_.

	
    * Que sólo pueda hacer _login_ desde la máquina necesaria.






	
  * Tener un directorio activo bien organizado para evitar agregar el nodo raíz. Si no, intentar optimizar el rendimiento con opciones como nss_initgroups o nss_getgrent_skipmembers.


﻿
