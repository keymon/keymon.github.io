---
author: keymon
comments: true
date: 2010-11-02 18:21:09+00:00
layout: post
slug: redescubriendo-restructuredtext
title: Redescubriendo reStructuredText
wordpress_id: 228
categories:
- fast-tip
- Misc
- Technical
tags:
- ReST
- reStructuredText
- rst
- wordpress
---

### reStructuredText


Estoy _"redescubriendo"_ el reStructuredText (también conocido como _rst_ o _ReST_).

Se trata de un lenguaje de markut (como HTML o SGML) pero _human friendly_. Similar al lenguaje de los wikis. Consulta [la pagina de reStructuredText en la wikipedia](http://en.wikipedia.org/wiki/ReStructuredText) para más detalles.

Parece tonto, pero la verdad es que se puede usar para muchas cosas, como _lenguaje franco_ para textos con formato simple: Documentación, sistema de ticketing, manuales...

Pero se puede ir más lejos:



	
  * entradas en blog (este post está echo con ReST), usando [este pequeño script en python](http://unmaintainable.wordpress.com/2008/03/22/using-rst-with-wordpress/)...Si vamos más lejos, [hay quien usa un gestor de versiones junto este script](http://tadhg.com/wp/2009/07/14/blog-workflow-with-restructuredtext/) !. Esto hace rondar una idea por la cabeza... ¿no molaría disponer de un blog en el que pudieras gestionar directamente con git?

	
  * O incluso presentaciones!, como comentan en esta página del propio docutils:

	
    * Original en Rest: [http://docutils.sourceforge.net/docs/user/slide-shows.txt](http://docutils.sourceforge.net/docs/user/slide-shows.txt)

	
    * En HTML normal: [http://docutils.sourceforge.net/docs/user/slide-shows.txt](http://docutils.sourceforge.net/docs/user/slide-shows.txt)

	
    * Como presentación en [S5](http://meyerweb.com/eric/tools/s5/): [http://docutils.sourceforge.net/docs/user/slide-shows.s5.html](http://docutils.sourceforge.net/docs/user/slide-shows.s5.html)





Y aquí vemos cómo generé este post (sin esta parte, para no hacer un post recursivamente infinito :)):

    
        cat <<EOF | ./rst2wp
        reStructuredText
        ----------------
    
        Estoy *"redescubriendo"* el reStructuredText_ (también conocido como *rst* o *ReST*).
    
        Se trata de un lenguaje de markut (como HTML o SGML) pero *human friendly*. Similar al lenguaje de los wikis. Consulta `la pagina de reStructuredText en la wikipedia <http://en.wikipedia.org/wiki/ReStructuredText>`_ para más detalles.
    
        Parece tonto, pero la verdad es que se puede usar para muchas cosas, como *lenguaje franco* para textos con formato simple: Documentación, sistema de ticketing, manuales...
    
        Pero se puede ir más lejos:
    
        - entradas en blog (este post está echo con ReST), usando `este pequeño script en python <http://unmaintainable.wordpress.com/2008/03/22/using-rst-with-wordpress/>`_...
    
        Si vamos más lejos, `hay quien usa un gestor de versiones junto este script <http://tadhg.com/wp/2009/07/14/blog-workflow-with-restructuredtext/>`_ !. Esto hace rondar una idea por la cabeza... ¿no molaría disponer de un blog en el que pudieras gestionar directamente con git?
    
        - O incluso presentaciones!, como comentan en esta página del propio docutils:
    
        * Original en Rest: http://docutils.sourceforge.net/docs/user/slide-shows.txt
    
        * En HTML normal: http://docutils.sourceforge.net/docs/user/slide-shows.txt
    
        * Como presentación en `S5 <http://meyerweb.com/eric/tools/s5/>`_: http://docutils.sourceforge.net/docs/user/slide-shows.s5.html::
    
    Y aquí vemos cómo generé este post (sin esta parte, para no hacer un post recursivamente infinito :))::
    
         cat <<EOF | ./rst2wp
        EOF
    
