---
author: keymon
comments: true
date: 2010-10-01 10:39:45+00:00
layout: post
slug: analisis-de-google-sobre-git-vs-mercurial
title: Analisis de Google sobre Git vs. Mercurial
wordpress_id: 220
categories:
- coding
- github
- Opinion
---

[En este post de Google ](http://code.google.com/p/support/wiki/DVCSAnalysis)se puede ver un análisis de Google sobre Git vs. Mercurial. En él se intentan justitificar de porqué no dan soporte a Git. El caso son aún más interesantes son los comentarios al artículo.

La verdad es que está claro que es un estudio muy sesgado. Muestran las diferencias entre ambos siempre considerando que lo que hace Git es peor. Además omiten muchas ventajas de Git (como que los commits son nodos que puede reorganizarse) o si las mencionan no loas muestran como ventajas (la posibilidad de usar cómodamente varios branchs).

Incluso incurren en falacias, como lo de que Git permite la perdida del historial con "git push --force". Para empezar es un --force, es decir, estás haciendo una chapuza poco habitual, y segundo que NO se pierde el historial.

En mi opinión, la principal desventaja, es el soporte de Windows que puede dar algún que otro problema...  y otra cosa que no se menciona: Que Git tiene una comunidad grande (y cada vez más grande) detrás, y además una comunidad de "Hackers". Y eso es generalmente bueno...

En conclusión, aunque a Google le duela, la realidad es que Git está creciendo de forma explosiva, cada dia decenas de proyectos en Google Code cuelgan el cartelito de "Proyecto hospedado en GitHub" en su sección de código.

No pasa nada si ellos no quieren aceptar esa realidad, al menos permiten hospedar proyectos, con su wiki, tracker y demás y mantener el código en github.com.
