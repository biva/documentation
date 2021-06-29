==========================================
OnlyOffice on the same server as Nextcloud
==========================================
 
This document explains how to install OnlyOffice on the same server as Nextcloud, with this configuration

- Debian/Apache
- Nextcloud
- OnlyOffice (Docker version, Document Server only) behind a proxy
- Let's encrypt for everything

Required specifications
------------------
- Apache2
- A working Nextcloud Instance with SSL
- Docker installed (see https://docs.docker.com/engine/install/)
- Let's encrypt (certbot) installed (see https://certbot.eff.org/instructions)

The examples below are working for Debian 10 Buster. Some modifications might be required for a different system. The URL used in this example are:

- Nextcloud: nextcloud.mydomain.com
- OnlyOffice: office.mydomain.com

Set up Apache and DNS
---------------------

Enable the required Apache modules:

::

    a2enmod proxy
    a2enmod proxy_wstunnel
    a2enmod proxy_http
    a2enmod headers

Create the virtual host for **office.mydomain.com**

::

  sudo nano /etc/apache2/sites-available/office.conf

Add the following lines in this file:

::

  <VirtualHost *:80>
  ServerName office.mydomain.com
  
  RewriteEngine on
  RewriteCond %{SERVER_NAME} =office.mydomain.com
  RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
  
  </VirtualHost>
  
Enable this virtual host:

::

  a2ensite office.conf
  service apache2 reload

Create the SSL version with certbot:

::

  sudo certbot --apache -d office.mydomain.com

This will automatically create :file:`/etc/apache2/sites-available/office-le-ssl.conf`. Edit this file

::

  sudo nano /etc/apache2/sites-available/office-le-ssl.conf

And add, at the end of the **<VirtualHost>** section:

::

  SetEnvIf Host "^(.*)$" THE_HOST=$1
  RequestHeader setifempty X-Forwarded-Proto https
  RequestHeader setifempty X-Forwarded-Host %{THE_HOST}e
  ProxyAddHeaders Off
  ProxyPass /.well-known !
  ProxyPassMatch (.*)(\/websocket)$ "ws://127.0.0.1:8006/$1$2"
  ProxyPass / http://127.0.0.1:8006/
  ProxyPassReverse / http://127.0.0.1:8006/


Enable this site:

::

  a2ensite office-le-ssl.conf
  service apache2 reload

Set the DNS
-----------

Let your system know that **office.mydomain.com** is itself:

::

  sudo nano /etc/hosts

and add/modify the first line to have something like this:

::

  127.0.0.1 localhost nextcloud.mydomain.com office.mydomain.com

Save and exit

Get the docker image
--------------------

::

  docker pull onlyoffice/documentserver

And launch the docker image (you can adapt "secret-key" and the port as you want)

::

  sudo docker run -i -t -d --name="onlyoffice" -p 127.0.0.1:8006:80 --restart=always -e JWT_ENABLED="true" -e JWT_SECRET="secret-key" onlyoffice/documentserver

Install the connector on Nextcloud
----------------------------------

Within Nextcloud apps (https://nextcloud.mydomain.com/settings/apps/), install and enable the app ONLYOFFICE.

Configure this app within the Settings (https://nextcloud.mydomain.com/settings/admin/onlyoffice)

- Address: office.mydomain.com
- Secret key: secret-key
- Click Save

Check status and test it
------------------------

Visit https://office.mydomain.com/healthcheck: it should show "true"


Additional optional steps
-------------------------
From previous attempts of installation, I had also done the following steps, but I'm not sure if they are required or not.

::

  sudo nano /var/www/html/nextcloud/config/config.php

Modify the file so that it includes:

::

  'trusted_domains' =>
    array (
      4 => 'office.mydomain.com',
    ),
    
and also the following:

::

  'allow_local_remote_servers' => true,
    'onlyoffice' =>
    array (
      'verify_peer_off' => true,
    ),

Credits and other installations possibilities
---------------------------------------------

These steps are largely inspired by https://help.nextcloud.com/t/howto-what-to-do-for-having-nextcloud-onlyoffice-on-the-same-host/33192 and https://arnowelzel.de/en/onlyoffice-in-nextcloud-the-current-status

OnlyOffice's team published a different method but it requires a clean installation: https://github.com/ONLYOFFICE/docker-onlyoffice-nextcloud/tree/feature/ssl
