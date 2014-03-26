---
author: keymon
comments: true
date: 2010-09-20 12:13:36+00:00
layout: post
slug: setting-puppetmaster-and-puppet-dashboard-with-passenger-in-a-debian-apache-server-using-puppet-user
title: Setting puppetmaster and puppet-dashboard with passenger in a Debian Apache
  server using 'puppet' user.
wordpress_id: 213
categories:
- puppet
- sysadmin
- Technical
- trick
tags:
- apache
- configuration
- efficiency
- linux
- passenger
- puppet
- rails
- trick
- www-data
---



I will describe my configuration of puppetmaster server and puppet  dashboard server running into the Debian's Apache installation, but:



	
  * Using a custom or different user, not www-data and root. This is  good to keep all puppet configuration and data with a different user  than www-data and root.

	
  * Using a custom configuration directory, not default apache  directory: /etc/apache2

	
  * You can issolate puppet server from the rest of apache  applications.


First I recommend you to read the official documentation:

	
  * [http://projects.reductivelabs.com/projects/puppet/wiki/Using_Passenger](http://projects.reductivelabs.com/projects/puppet/wiki/Using_Passenger).

	
  * [http://projects.puppetlabs.com/projects/puppet/wiki/Using_Mongrel](http://projects.puppetlabs.com/projects/puppet/wiki/Using_Mongrel)


It is supposed that you have a running puppet installation.


### <!-- more -->




### Install[ ](http://trac.dclinuxapps1.caixagalicia.cg/wiki/SandBox?version=12#Install)


You should first install all needed packages. Be sure of use all last  versions, in my case:



	
  * ruby 1.8.7

	
  * libactiverecord 2.3.5-1

	
  * libmysql-ruby 2.7.4-1

	
  * librack-ruby 1.1


I use Debian with lenny-backports so:

    
    apt-get -t lenny-backports install apache2 libapache2-mod-passenger rails librack-ruby libmysql-ruby
    




### Set  apache configuration directory to a different location[ ](http://trac.dclinuxapps1.caixagalicia.cg/wiki/SandBox?version=12#Setapacheconfigurationdirectorytoadifferentlocation)


I have my puppet configuration:



	
  * puppet config and data path: /cgx1/puppet,

	
  * server user and group: puppet


I set all the paths relative to /cgx1/puppet

Will install all apache configuration on /cgx1/puppet/conf/apache2

First I clone the apache configuration:

    
    cd /cgx1/puppet/conf
    cp -Ra /cgx1/puppet/conf/apache2 .
    # clean the directories with other site configuration
    cd conf.d; rm nagios3.conf gitweb awstats.conf  zabbix.conf; cd ..
    # Remove old sites
    rm sites-*/
    # Remove the mods-available and link all to the /etc/apache2/mods-available/
    cd mods-available/; ln -s /etc/apache2/mods-available/* .; cd ..
    # Remove unneded plugins:
    cd mods-enabled; rm php5.load php5.conf python.load; cd ..
    


Then we have to fix the configuration to adapt all the paths:



	
  * /etc/apache2 => /cgx1/puppet/conf/apache2

	
  * /var/lock => /cgx1/puppet/var/lock

	
  * /var/log => /cgx1/puppet/var/log

	
  * /var/run => /cgx1/puppet/var/run



    
    # First create the directories
    mkdir -p \
    	/cgx1/puppet/conf/apache2 \
    	/cgx1/puppet/var/lock/apache2 \
    	/cgx1/puppet/var/log/apache2 \
    	/cgx1/puppet/var/run/apache2 
    
    # update apache2.conf
    sed 's|/etc/apache2|/cgx1/puppet/conf/apache2|;
    	 s|/var/lock|/cgx1/puppet/var/lock|;
    	 s|/var/log|/cgx1/puppet/var/log|;' \
    	 s|/var/run|/cgx1/puppet/var/run|;' \
         -i apache2.conf
    
    # Adapt ssl.conf => ssl-puppet.conf
    sed 's|/var/run|/cgx1/puppet/var/run|;' < mods-available/ssl.conf  > mods-available/ssl-puppet.conf 
    rm mods-enabled/ssl.conf
    ln -s ../mods-available/ssl-puppet.conf mods-enabled/
    


Then, I use a wrapper to "clone" the apache2ctl, a2enmod, a2dismod,  a2ensite and a2dissite scripts. This script will set all needed  variables to make these scripts work in the new location and will force  the user to 'puppet' with sudo.

    
    # pwd => /cgx1/puppet/conf/apache2
    mkdir bin
    
    cat <<"EOF" > bin/puppet_apache_wrapper
    #!/bin/sh
    
    # Search the real script
    SCRIPT_NAME=$(basename $0 | sed 's/puppet_//')
    REAL_SCRIPT=$(which $SCRIPT_NAME 2> /dev/null)
    
    if [ -z "$REAL_SCRIPT" ]; then
            echo "$0: $SCRIPT_NAME not found in \$PATH" 1>&2
            exit 1;
    fi
    
    # Set the new location
    export APACHE_CONFDIR=/cgx1/puppet/conf/apache2
    export APACHE_ENVVARS=/cgx1/puppet/conf/apache2/envvars
    
    [ -f "$APACHE_ENVVARS" ] && . $APACHE_ENVVARS
    
    # Run it as $APACHE_RUN_USER
    ACTUAL_USER=$(id -u -n)
    if [ "$ACTUAL_USER" == "$APACHE_RUN_USER" ]; then
            exec $REAL_SCRIPT "$@"
    else
            exec sudo -u puppet -p "User password: " $0 "$@"
    fi
    
    EOF
    
    chmod +x bin/puppet_apache_wrapper
    
    # And the create the scripts 
    ln -s puppet_apache_wrapper bin/puppet_apache2ctl
    ln -s puppet_apache_wrapper bin/puppet_a2dismod 
    ln -s puppet_apache_wrapper bin/puppet_a2dissite 
    ln -s puppet_apache_wrapper bin/puppet_a2enmod  
    ln -s puppet_apache_wrapper bin/puppet_a2ensite 
    
    # Adapt envvars - default environment variables for apache2ctl
    
    cat <<"EOF"
    # envvars - default environment variables for apache2ctl
    
    # Since there is no sane way to get the parsed apache2 config in scripts, some
    # settings are defined via environment variables and then used in apache2ctl,
    # /etc/init.d/apache2, /etc/logrotate.d/apache2, etc.
    export APACHE_RUN_USER=puppet
    export APACHE_RUN_GROUP=puppet
    export APACHE_PID_FILE=/cgx1/puppet/var/run/apache2-puppet.pid
    
    export APACHE_STATUSURL=-https://localhost:8140/server-status
    export APACHE_RUN_DIR=/cgx1/puppet/var/run/
    
    export APACHE_ARGUMENTS="-d /cgx1/puppet/conf/apache2 \
                             -f /cgx1/puppet/conf/apache2/apache2.conf"
    EOF
    
    


Finally, we adapt the ports that will be started:

    
    cat<<EOF > ports.conf
    NameVirtualHost *:8140
    Listen 8140
    NameVirtualHost *:3000
    Listen 3000
    EOF
    


Now we can start the server, test configurations... but probably you  want create the virtual host first

    
    ./bin/puppet_apache2ctl configtest
    ./bin/puppet_apache2ctl start
    




### Puppetmaster and  dashboard virtualhosts[ ](http://trac.dclinuxapps1.caixagalicia.cg/wiki/SandBox?version=12#Puppetmasteranddashboardvirtualhosts)


Fisrt we have to create the "public" configuration for librack for the  puppetmaster. This step is described in official wiki:

    
    # Create rack/public and config.ru
    cd /cgx1/puppet
    
    mkdir -p conf/rack/public/
    
    cat <<"EOF" > conf/rack/config.ru
    #-------------------------------------------------
    # This options if puppet is out of default the RUBYLIB:
    # $:.unshift('/opt/puppet/lib')
    $:.unshift('/usr/local/lib/ruby')
    $:.unshift('/usr/local/lib/ruby/1.8')
    $:.unshift('/usr/local/lib/ruby/vendor_ruby/1.8')
    $:.unshift('/usr/local/lib/ruby/site_ruby/1.8')
    
    $0 = "master"
    
    #-------------------------------------------------
    # Extra arguments for the puppetmaster
    ARGV << "--rack"
    
    # if you want debugging:
    # ARGV << "--debug"
    
    # Different configuration path 
    ARGV << "--config=/cgx1/puppet/conf/puppet.conf"
    require 'puppet/application/master'
    
    #-------------------------------------------------
    # we're usually running inside a Rack::Builder.new {} block,
    # therefore we need to call run *here*.
    run Puppet::Application[:master].run
    EOF
    


And my site configuration file:

    
    cd /cgx1/puppet/conf/apache2/
    cat <<"EOF" > sites-available/puppetmaster-passenger.conf
    # Configuration based on
    #   http://projects.reductivelabs.com/projects/puppet/wiki/Using_Passenger
    
    # Puppetmaster listens on: 8140
    #Listen 8140
    
    <VirtualHost *:8140>
        # This is a SSL hots
        SSLEngine on
        SSLCipherSuite SSLv2:-LOW:-EXPORT:RC4+RSA
    
        # Puppetmaster certificates
        SSLCertificateFile /cgx1/puppet/var/ssl/certs/myhost.mycompany.com.pem
        SSLCertificateKeyFile /cgx1/puppet/var/ssl/private_keys/myhost.mycompany.com.pem
        SSLCertificateChainFile /cgx1/puppet/var/ssl/ca/ca_crt.pem
        SSLCACertificateFile /cgx1/puppet/var/ssl/ca/ca_crt.pem
    
        # Using the technique described in http://projects.puppetlabs.com/projects/puppet/wiki/Using_Mongrel
        # mkdir /cgx1/puppet/var/ssl/server/crl
        # cd /cgx1/puppet/var/ssl/server/crl; ln -s /cgx1/puppet/var/ssl/ca/ca_crl.pem `openssl crl -in /cgx1/puppet/var/ssl/ca/ca_crl.pem -hash -noout`.0
        SSLCARevocationPath     /cgx1/puppet/var/ssl/server/crl
    
        # SSLVerifyClient: by enabling require, you basically say:
        #  "the client is already suppose to have a certificate that I could verify. "
        #  If certificate is not signed in puppetmaster you will get the error:
        #    err: Could not request certificate: SSL_connect returned=1 errno=0 state=SSLv3 read finished A: sslv3 alert handshake failure
        # With optional you will simulate the behaviour of an default standalone puppetmaster (without apache).
        SSLVerifyClient optional
        #SSLVerifyClient require
        SSLVerifyDepth  1
        SSLOptions +StdEnvVars
    
        # Use this to allow SSL signing
        # The following client headers allow the same configuration to work with Pound.
        RequestHeader set X-SSL-Subject %{SSL_CLIENT_S_DN}e
        RequestHeader set X-Client-DN %{SSL_CLIENT_S_DN}e
        RequestHeader set X-Client-Verify %{SSL_CLIENT_VERIFY}e
    
        # Logs
        ErrorLog /cgx1/puppet/var/log/apache2/puppetmaster-error.log
        LogLevel warn
        CustomLog /cgx1/puppet/var/log/apache2/puppetmaster-access.log combined
        CustomLog /cgx1/puppet/var/log/apache2/puppetmaster-ssl_request.log \
                 "%t %h %{SSL_PROTOCOL}x %{SSL_CIPHER}x \"%r\" %b"
    
    
        # Passenger configuration
        RackAutoDetect On
        DocumentRoot /cgx1/puppet/conf/rack/public/
        <Directory /cgx1/puppet/conf/rack>
            Options None
            AllowOverride None
            Order allow,deny
            allow from all
        </Directory>
    </VirtualHost>
    EOF 
    ln -s ../sites-available/puppetmaster-passenger.conf sites-enabled/
    


Puppet dashboard configuration is easier:

    
    cat <<"EOF" > sites-available/dashboard-passenger.conf
    # Dashboard configuration
    <VirtualHost *:3000>
        RackAutoDetect On
        DocumentRoot /opt/puppet-dashboard/public/
    
        # Set the development to serve
        SetEnv RAILS_ENV development
    
        <Directory /opt/puppet-dashboard/public/>
            Options None
            AllowOverride AuthConfig
            Order allow,deny
            Allow from all
        </Directory>
    
        ServerSignature On
        # Logs
        ErrorLog /cgx1/puppet/var/log/apache2/dashboard-error.log
        LogLevel warn
        CustomLog /cgx1/puppet/var/log/apache2/dashboard-access.log combined
        CustomLog /cgx1/puppet/var/log/apache2/dashboard-ssl_request.log \
                 "%t %h %{SSL_PROTOCOL}x %{SSL_CIPHER}x \"%r\" %b"
    
    </VirtualHost>
    EOF
    ln -s ../sites-available/dashboard-passenger.conf sites-enabled/
    


Now you restart and test the configuration:

    
    /cgx1/puppet/conf/apache2/bin/puppet_apache2ctl configtest
    /cgx1/puppet/conf/apache2/bin/puppet_apache2ctl restart
    




### Putting the  points to the i's and crossing the t's[
](http://trac.dclinuxapps1.caixagalicia.cg/wiki/SandBox?version=12#Puttingthepointstotheisandcrossingthets)


Logrotate:

    
    cat <<"EOF" > /cgx1/puppet/conf/apache2/puppet-passenger.logrotate
    /cgx1/puppet/var/log/apache2/*.log {
            weekly
            missingok
            rotate 52
            compress
            delaycompress
            notifempty
            create 640 puppet puppet
            sharedscripts
            postrotate
                    if [ -f "`. /cgx1/puppet/conf/apache2/envvars ; echo ${APACHE_PID_FILE:-/cgx1/puppet/var/run/apache2-puppet.pid}`" ]; then
                            /cgx1/puppet//conf/apache2/bin/puppet_apache2ctl graceful > /dev/null
                    fi
            endscript
    }
    EOF
    ln -s /cgx1/puppet/conf/apache2/puppet-passenger.logrotate /etc/logrotate.d 
    



