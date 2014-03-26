---
author: keymon
comments: true
date: 2009-09-08 10:40:17+00:00
layout: post
slug: extendiendo-el-runtime-de-java-con-nuevos-idiomas-del-www-elrincondelprogramador-com
title: Extendiendo el Runtime de Java con nuevos idiomas (del www.elrincondelprogramador.com)
wordpress_id: 7
categories:
- coding
---

Hace tiempo publiqué en [www.elrincondelprogramador.com](http://www.elrincondelprogramador.com/default.asp?pag=articulos/leer.asp&id=70) un articulillo (creo que el único, tristemente) comentando el tema de [cómo extender los locales para que soporten gl_ES](http://www.elrincondelprogramador.com/default.asp?pag=articulos/leer.asp&id=70). Lo cierto es que el articulo se extiende a cualquier locale, por supuesto.

Me fastidió mucho el tema porque [Java comentaba ](http://bugs.sun.com/bugdatabase/view_bug.do?bug_id=5104387)que no daban soporte porque no lo consideraban un idioma oficial... La culpa es de los gallegos, que no defendemos lo nuestro, porque el soporte en Vasco o Catalán si está.

<!-- more -->


Extendiendo el Runtime de Java con nuevos idiomas




_ [Héctor Rivas ](http://www.elrincondelprogramador.com/autores.asp?id=15) _






Cuando desarrollamos una aplicación con soporte para              internacionalización, solemos (y debemos) confiar el formateo y              análisis de fechas y valores monetarios a clases que actúen de forma             independiente del lenguaje y país actual. Estas clases están             incluidas en la propia plataforma [Java1](http://www.elrincondelprogramador.com/default.asp?pag=articulos/leer.asp&id=70#probado):              [DateFormat](http://java.sun.com/j2se/1.4.2/docs/api/java/text/DateFormat.html) y [ NumberFormat](http://java.sun.com/j2se/1.4.2/docs/api/java/text/NumberFormat.html).

Por desgracia, [el número de locales soportados              por estas clases es limitado2](http://www.elrincondelprogramador.com/default.asp?pag=articulos/leer.asp&id=70#locales) y si intentamos formatear una              fecha con un _locale_ desconocido, obtendremos el texto              formateado con el _locale_ por defecto del sistema.

Este problema se agrava al no estar documentada ni definida ninguna             forma de extender el sistema con nuevos _locales_, obligando al             programador a escribir código dependiente del idioma o utilizar             clases propias para el formateo de fechas. Además, numerosas APIs y             frameworks, como [Struts](http://struts.apache.org/), JSTL, SWING, SWT, etc.             utilizan el sistema de internacionalización de Java, comportamiento             que en pocas ocasiones podemos modificar, anulando cualquier solución             alternativa.

Podemos comprobar este comportamiento con este sencillo código de              prueba:

    
    import java.io.*;
    import java.text.*;
    import java.util.*;
    
    public class HelloGalician  {
    
        public static void main (String Argv[]) {
    
            Locale locale = new Locale("gl", "ES");
    
            DateFormat fmt = DateFormat.getDateTimeInstance(
                DateFormat.LONG, DateFormat.LONG, locale);
    
            fmt.setTimeZone(TimeZone.getDefault());
    
            System.out.println("En galego:");
            System.out.println("\u00a1Hola Mundo!");
            System.out.println(fmt.format(new Date()));
        }
    }


Si compilamos y ejecutamos este código con un _runtime_ sin            soporte de gl_ES la salida es:

    
    En galego:
    ¡Hola Mundo!
    January 2, 2005 4:44:19 PM CET


Pero no todo está perdido, existe una forma de agregar nuestros              propios locales de una forma que, aun siendo poco ortodoxa,              funciona correctamente y permite mantener la independencia del              código respecto al idioma.

Si analizamos el contenido de los ficheros .jar de la              plataforma (jar –tvf), y en particular los ficheros              jre/lib/rt.jar y jre/lib/ext/localedata.jar , observamos que para todos los locale soportados existen las              siguientes clases:

    
    ...
    sun/text/resources/LocaleElements_es_MX.class
    sun/text/resources/LocaleElements_es_ES.class
    sun/text/resources/LocaleElements_es_CA.class
    ...
    sun/text/resources/DateFormatZoneData_es.class
    sun/text/resources/DateFormatZoneData_et.class
    sun/text/resources/DateFormatZoneData_fi.class
    ...


Seguramente estas son las clases que debemos añadir. Cabe              preguntarse qué son estas clases. [Buceando por              internet3](http://www.elrincondelprogramador.com/default.asp?pag=articulos/leer.asp&id=70#buceando) pronto descubrimos que ambas clases heredan de              java.util.ListResourceBundle, que son utilizadas por              DateFormat para el formateo de las fechas y que              efectivamente deben existir ambas clases por cada _locale_ formateado.

Sólo nos falta determinar los valores que deben contener estos              ListResourceBundle. Decompilar sería una opción si no              lo prohibiese la licencia de SUN, pero por suerte también resulta              sencillo encontrar en internet una implementación de las mismas o,              simplemente, podemos consultar el método getContents() de dichas clases para el locale por defecto:

    
    import java.util.ResourceBundle;
    import java.util.Locale;
    import java.util.Enumeration;
    
    public class ListResourcePrint {
    
        static String toString(Object o) {
    
            if (o instanceof Object []) {
                Object array [] = (Object []) o;
                String str = "[";
                for (int i = 0; i < array.length; i++) {
                    if (i>0) str+=", ";
                    str+=toString(array[i]);
                }
                str += "]";
                return str;
            } else if (o instanceof String) {
                return "\""+o.toString()+"\"";
            } else {
                return o.toString();
            }
        }
    
        static void printResource(ResourceBundle l) {
            for (Enumeration i = l.getKeys();
                i.hasMoreElements(); ) {
    
                String key = (String) i.nextElement();
                Object value = l.getObject(key);
                if (value != null)
                    System.out.println(key + " = "
                        + toString(value));
                else
                    System.out.println(key + " = null");
            }
        }
    
        public static final void main(String args[]) {
    
            System.out.println("LocaleElements:");
            printResource(ResourceBundle.getBundle(
                "sun.text.resources.LocaleElements",
                    Locale.getDefault()));
            System.out.println("DateFormatZoneData:");
            printResource(ResourceBundle.getBundle(
                "sun.text.resources.LocaleElements",
                    Locale.getDefault()));
        }
    }


Es ahora cuando podemos implementar nuestras propias clases              LocaleElements_gl_ES y               DateFormatZoneData_gl_ES:

    
    package sun.text.resources;
    
    import java.util.ListResourceBundle;
    
    public class LocaleElements_gl extends ListResourceBundle {
    
        /**
         * Overrides ListResourceBundle
         */
        public Object[][] getContents() {
            return new Object[][] {
                // locale id based on iso codes
                { "LocaleString", "gl_ES" }, 
    
                (..)
    
                { "MonthNames",
                    new String[] {
                        "xaneiro", // january
                    (..)
                },
                { "CollationElements", "@" }
            };
        }
    }


Una vez implementadas y compiladas, debemos agregarlas a la              plataforma. El problema ahora es que el paquete              sun.text.resources es un paquete protegido, y la              máquina virtual de java no las cargará desde el _classpath_ de usuario.

Para que las cargue, debemos construir un fichero .jar con nuestras clases y copiarlo en el _classpath_ del sistema,              concretamente en el directorio de extensiones de Java (normalmente              $JAVA_HOME/jre/lib/ext).

Una vez instalado, ya estamos listos para probar nuestro nuevo              _locale_ simplemente ejecutando el código anterior:

    
    En galego:
    ¡Hola Mundo!
    2 de xaneiro de 2005 16:46:52 CET


Tenemos ahora una máquina virtual parcheada que soporta nuestro              nuevo idioma en la que podemos emplear el API de              internazionalización de Java y cualquier otro API o framework que              emplee internazionalización.

De todos modos esta solución plantea algunos problemas. Las clases              LocaleElements y DateFormatZoneData no              entran dentro del API de Java, y su funcionamiento puede ser              modificado en futuras versiones de la plataforma. Además, no es              posible extender los _locales_ a nivel de aplicación, sino que              la máquina virtual debe parchearse instalando un paquete en el              directorio de extensiones, al cual no siempre tenemos acceso y              además complica la instalación y distribución de la aplicación.

La solución más elegante sería, sin duda, que el propio API de Java              permitiese registrar nuevos locale aparte de los ofrecidos por la              plataforma. Mientras tanto, habrá que limitarse a pequeños trucos              como éste y a solicitar el soporte de nuestro _locale_ a los              fabricantes.






**1** Todo lo comentado a continuación ha sido realizado y probado sobre              Java 2 Platform Standard Edition (J2SE) de Sun.         [»ir al texto](http://www.elrincondelprogramador.com/default.asp?pag=articulos/leer.asp&id=70#noteprobado)

**2** Podemos conocer los _locale_ soportados por el runtime              consultando la documentación ([lista de              locale soportados por j2se 1.4.2](http://java.sun.com/j2se/1.4.2/docs/guide/intl/locale.doc.html)) o consultando el método              [Locale.getAvailableLocales()](http://java.sun.com/j2se/1.4.2/docs/api/java/util/Locale.html#getAvailableLocales%28%29).         [»ir al texto](http://www.elrincondelprogramador.com/default.asp?pag=articulos/leer.asp&id=70#notelocales)

**3** Por [unas](http://www.google.com/search?q=import+DateFormatZoneData) y por [otras](http://www.google.com/search?q=LocaleElements+import), y              como siempre, _Google_ nos puede resultar de ayuda.         [»ir al texto](http://www.elrincondelprogramador.com/default.asp?pag=articulos/leer.asp&id=70#notebuceando)




## Referencias





	
  * [Javadoc de la clase DateFormat](http://java.sun.com/j2se/1.4.2/docs/api/java/text/DateFormat.html)

	
  * [Javadoc de la clase NumberFormat](http://java.sun.com/j2se/1.4.2/docs/api/java/text/NumberFormat.html)

	
  * [Javadoc del método Locale.getAvailableLocales()](http://java.sun.com/j2se/1.4.2/docs/api/java/util/Locale.html#getAvailableLocales%28%29)

	
  * [Locales soportados por J2SE 1.4.2](http://java.sun.com/j2se/1.4.2/docs/guide/intl/locale.doc.html)


