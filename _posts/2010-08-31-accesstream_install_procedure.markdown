---
author: keymon
comments: true
date: 2010-08-31 11:50:45+00:00
layout: post
slug: accesstream_install_procedure
title: Accesstream software installation procedure
wordpress_id: 188
categories:
- fast-tip
- linux/unix
- Misc
- sysadmin
tags:
- accesstream
- linux
- sysadmin
---

I have been asked to install [AccessStream application ](http://www.accesstream.com/)on linux to test it.


But this application has not ANY documentation or install procedure. At the end I managed to install it, here is the procedure.

<!-- more -->


#### Download[
](http://trac.dclinuxapps1.caixagalicia.cg/wiki/InstalacionAccesstream?version=2#Download)





	
  * [http://sourceforge.net/projects/accesstream/files/](http://sourceforge.net/projects/accesstream/files/)

	
    * accesstream-postgres-testdata-backup-1.2.0.backup

	
    * accesstream-webapp-1.2.0.zip






	
  * Tomcat: [http://apache.rediris.es//tomcat/tomcat-7/v7.0.2-beta/bin/apache-tomcat-7.0.2.tar.gz](http://apache.rediris.es//tomcat/tomcat-7/v7.0.2-beta/bin/apache-tomcat-7.0.2.tar.gz)



	
  * jdk 1.6




#### Instalation[
](http://trac.dclinuxapps1.caixagalicia.cg/wiki/InstalacionAccesstream?version=2#Instalation)


Tomcat:

    
    cd /opt/
    tar -xvzf apache-tomcat-7.0.2.tar.gz
    cd apache-tomcat-7.0.2/
    


Add rol manager conf/tomcat-users.xml and user tomcat

    
      <role rolename="manager-gui"/>
      <user username="tomcat" password="werfvcxs" roles="tomcat,manager-gui"/>
    




#### Deploy[
](http://trac.dclinuxapps1.caixagalicia.cg/wiki/InstalacionAccesstream?version=2#Deploy)





	
  * Copy accesstream-webapp-1.2.0.zip *.war to  /opt/apache-tomcat-7.0.2/webapps

	
  * Start tomcat:

    
    ./bin/catalina.sh start
    






	
  * Must uncompress the  WEB-INF/lib/config_classes.jar in classes of web-admin and web-idgateway  (????)

    
    cd /opt/apache-tomcat-7.0.2/webapps/web-admin/WEB-INF/classes/
    /opt/java/bin/jar -xvf ../lib/config_classes.jar
    cd /opt/apache-tomcat-7.0.2/webapps/web-idgateway/WEB-INF/classes/
    /opt/java/bin/jar -xvf ../lib/config_classes.jar
    







#### Database[
](http://trac.dclinuxapps1.caixagalicia.cg/wiki/InstalacionAccesstream?version=2#Database)


Install postgresql-server. Update /var/lib/pgsql/data/pg_hba.conf:

    
    # TYPE  DATABASE    USER        CIDR-ADDRESS          METHOD
    
    # "local" is for Unix domain socket connections only
    local   all         postgres                               ident sameuser
    # IPv4 local connections:
    host    all         postgres         127.0.0.1/32          ident sameuser
    # IPv6 local connections:
    host    all         postgres         ::1/128               ident sameuser
    
    # "local" is for Unix domain socket connections only
    local   all         all                               md5 sameuser
    # IPv4 local connections:
    host    all         all         127.0.0.1/32          md5 sameuser
    # IPv6 local connections:
    host    all         all         ::1/128               md5 sameuser
    


Using a Postgres 8.4 pg_restore (may be you need execute this in other  host), we generate the .sql from the dump downloaded:

    
    pg_restore  accesstream-postgres-testdata-backup-1.2.0.backup  > accesstream-postgres-testdata-backup-1.2.0.backup.sql
    


Must create DB and user (data in  web-admin.war/WEB-INF/classes/spring/datasource/xapool.xml):



	
  * DB: accesstream

	
  * user: accesstream

	
  * pass: idvs


In postgres account:

    
    su - postgres
    createdb accesstream
    createuser accesstream
    createuser public
    psql <<EOF
    alter user accesstream with encrypted password 'idvs';
    grant all privileges on database accesstream to accesstream;
    EOF
    psql accesstream -f accesstream-postgres-testdata-backup-1.2.0.backup.sql
    




#### Restart[
](http://trac.dclinuxapps1.caixagalicia.cg/wiki/InstalacionAccesstream?version=2#Restart)





	
  * Restart tomcat:

    
    ./bin/catalina.sh stop
    killall java
    rm logs/*
    ./bin/catalina.sh start
    





