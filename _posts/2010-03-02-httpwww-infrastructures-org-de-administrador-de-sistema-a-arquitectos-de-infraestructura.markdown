---
author: keymon
comments: true
date: 2010-03-02 16:59:41+00:00
layout: post
slug: httpwww-infrastructures-org-de-administrador-de-sistema-a-arquitectos-de-infraestructura
title: http://www.infrastructures.org/, de "administrador de sistema" a "arquitectos
  de infraestructura"
wordpress_id: 22
categories:
- references
- sysadmin
---

Buscando sobre el tema de la gestión y despliegue automático de la configuración de una red de sistemas, me encontré con esta página:

http://www.infrastructures.org/

Me gusta la filosofía y sirve como base para establecer e implatntar el decálogo que nos hará pasar de "administradores de sistema" a "arquitectos de infraestructura".

<!-- more -->

Me agrada encontrarme con comentarios que he extraído de mi propia experiencia pero que nunca vi reflejados en ningún sitio. Por ejemplo:



	
  * _"Indirect mounts are       more flexible than direct mounts, and are usually less buggy.       If a vendor's application insists that it must live at       /usr/appname and you want to keep that application on a       central server, resist the temptation to simply mount or       direct automount the directory to /usr/appname."_

	
  * _"We tend to use hostname aliases in DNS, and in our scripts and       configuration files, to denote which hosts currently offer       which services.  This way, we don't have to edit scripts       when a service moves from one host to another. "_

	
  * etc.


Sin duda, mi gran problema y lastre es el tema de la gestión de la configuración... seguiré investigando
