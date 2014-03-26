---
author: keymon
comments: true
date: 2010-08-19 16:04:07+00:00
layout: post
slug: recuperar-ficheros-de-texto-en-xfs
title: Recuperar ficheros de texto en xfs
wordpress_id: 173
categories:
- Misc
- Personal
tags:
- backup
- restore
- xfs
---

1º Moraleja: Hay que hacer backup.

Hoy por accidente he borrado todos los ficheros de un documento en latex. Para variar, y como "en casa de herrero cuchillo de palo", pues no tenía backup. Mi filesystem es xfs y tras mucho buscar en google sólo encontré algo sobre una utilidad de windows... demasiado complicado...

Pero al final recuperé los ficheros ¿cómo? Simple: los importantes eran 2 o 3 ficheros latex... abrí el dispositivo / con "less /dev/sda6" y busqué a saco cadenas que sabía que estaban en el documento, como mi nombre. Luego copié y pegué...

1º Moraleja: Hay que hacer backup.

2º Moraleja: Hay que hacer backup.

3º Moraleja: Hay que hacer backup.

2º Moraleja: La solución simple es muchas veces mucho más efectiva :)
