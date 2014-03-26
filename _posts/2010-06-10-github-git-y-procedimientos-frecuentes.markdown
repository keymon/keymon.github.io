---
author: keymon
comments: true
date: 2010-06-10 12:31:45+00:00
layout: post
slug: github-git-y-procedimientos-frecuentes
title: Github, git y procedimientos frecuentes
wordpress_id: 129
categories:
- coding
- fast-tip
- github
- references
- script
tags:
- coding
- git
- github
- gpl
- share
- source
- tip
- trick
---

Quiero presentar github ([http://github.com/](http://github.com/)), un site gratuito para  hosting de código fuente (publico o privado) en internet. Con todas las  ventajas de git, pero "en la nube" y gratis :).

Lo he estado probando y parece que cubre perfectamente mis necesidades:



	
  * Puedo tener un repositorio online sin límites.

	
  * Permite consultar y enlazar los fuentes, con iluminación de  síntaxis o en modo 'raw'

	
  * **Puede ser accesible desde detrás de un proxy que sólo  soporta HTTP/HTTPS**: Esto es esencial para poder trabajar desde  la oficina. Para ello tienen un ssh escuchando en el 443 del nombre  ssh.github.com.


<!-- more -->En este post voy a explicar brevemente los pasos para usarlo. No  pretendo crear un manual exhaustivo, sólo tener una referencia de lo que  voy haciendo:

	
  1. Instalamos y configuramos git. Para configurarlo:

    
    git config --global user.name "Keymon"
    git config --global user.email keymon@gmail.com
    






	
  1. Creamos una cuenta de github ([https://github.com/signup/free](https://github.com/signup/free)).



	
  1. Configuramos la conexión para SSH para trabajar con  github via SSH. Para eso:

	
    1. Creamos una clave asimétrica SSH. Podemos usar la nuestra de  siempre o crear una específica. Yo creo una distinta:

    
    ssh-keygen -f  ~/.ssh/id_rsa.github
    




	
    2. Configuramos el ssh en el ~/.ssh/config, para que use la  clave asimétrica y, en mi caso, el proxy y el puerto de 443 de  ssh.github.com.  Yo recomiendo usar ~/.ssh/config para separar la configuración de la  conexión de la configuración del repositorio.







> 

>
>> 

>>
>>> Lo curioso es que a este servidor SSH es que el usuario con el que  conecta es 'git', identifica al usuario por la clave asimétrica.
>> 
>> 

> 
> 





> 

>
>> 

>>
>>> Para el proxy, yo tengo configurado un ISA Client en el PC, que hace  transparente la configuración del proxy. Pero se podría definir un [ProxyCommand con el connect.c](http://bent.latency.net/bent/git/goto-san-connect-1.85/src/connect.html). :

>>>     
>>>     Host *
>>>     
>>>     Host gitproxy
>>>       Port 443
>>>       HostName ssh.github.com
>>>       ProxyCommand /home/hector/bin/connect -H proxy:80 %h %p
>>>       IdentityFile /home/hector/.ssh/id_rsa.github
>>>     
>>>     Host ssh.github.com github.com
>>>       Port 443
>>>       HostName ssh.github.com
>>>       IdentityFile /home/hector/.ssh/id_rsa.github
>>>     
>>> 
>>> 

>> 
>> 

> 
> 






	
  1. Creamos un nuevo repositorio en la página, _clickando_ en _Dashboard > Create a Repository_. Le damos un nombre al  repositorio y ya nos mostrará un pequeño manual de qué hacer después:

    
    Next steps:
    
      mkdir prueba
      cd prueba
      git init
      touch README
      git add README
      git commit -m 'first commit'
      git remote add origin git@github.com:keymon/prueba.git
      git push origin master
    
    Existing Git Repo?
    
      cd existing_git_repo
      git remote add origin git@github.com:keymon/prueba.git
      git push origin master
    







> 

>
>> Por ejemplo, yo creé un repositorio en local...:

>>     
>>     mkdir prueba; cd prueba
>>     git init
>>     touch README
>>     git add README
>>     git commit -m "Initial commit"
>>     
>> 
>> 
y luego lo añado.

>>     
>>     $ git remote add origin git@github.com:keymon/prueba.git
>>     $ git push origin master
>>     Counting objects: 3, done.
>>     Writing objects: 100% (3/3), 218 bytes, done.
>>     Total 3 (delta 0), reused 0 (delta 0)
>>     To git@github.com:keymon/prueba.git
>>      * [new branch]      master -> master
>>     
>> 
>> 

> 
> 



Referencias:



	
  * [http://github.com](http://github.com/)

	
  * [http://help.github.com/](http://help.github.com/)

	
  * [Access GitHub repositories from work (take that,  firewall!).](http://blog.codeslower.com/2008/8/Using-PuTTY-and-SSL-to-securely-access-GitHub-repositories-via-SSH)

	
  * [http://bent.latency.net/bent/git/goto-san-connect-1.85/src/connect.html](http://bent.latency.net/bent/git/goto-san-connect-1.85/src/connect.html)




## Tareas frecuentes[
](http://trac.dclinuxapps1.caixagalicia.cg/wiki/SandBox?version=7#Tareasfrecuentes)




### Crear un nuevo repositorio[
](http://trac.dclinuxapps1.caixagalicia.cg/wiki/SandBox?version=7#Crearunnuevorepositorio)



    
    mkdir prueba
    cd prueba
    git init
    touch README
    git add README
    git commit -m 'first commit'
    git remote add origin git@github.com:keymon/prueba.git
    git push origin master
    




### Crear un nuevo  repositorio desde uno existente[
](http://trac.dclinuxapps1.caixagalicia.cg/wiki/SandBox?version=7#Crearunnuevorepositoriodesdeunoexistente)



    
    cd existing_git_repo
    git remote add origin git@github.com:keymon/prueba.git
    git push origin master
    




### Clonar el repositorio  git de otra persona[
](http://trac.dclinuxapps1.caixagalicia.cg/wiki/SandBox?version=7#Clonarelrepositoriogitdeotrapersona)


(de [http://bent.latency.net/bent/git/goto-san-connect-1.85/src/connect.html](http://bent.latency.net/bent/git/goto-san-connect-1.85/src/connect.html))

You just want to patch some else’s repo:

    
    This is really simple. Here are the steps:
    
       1. Go to github and click the “fork” button.
       2. git clone git://github.com/stevenbristol/lovd-by-less.git
       3. cd lovd-by-less
       4. Make your cahnges
       5. git status
       6. git commit -a
       7. git push
       8. go back to git hub and click the “pull request” button.
    
    I’m guessing that by now this is really clear.
    
       1. Step 1 will create you own fork of the repo where you can make your changes. Click this on the person’s page you want to fork from.
       2. Steps 2-7 you should understand by now. (I hope. :)
       3. Step 8 will send a message to the person notifing them that you have something for them to see. Click this from
