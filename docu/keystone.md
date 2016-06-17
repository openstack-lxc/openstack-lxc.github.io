---
layout: blog
title: Keystone
navbar:
  - Documentación
menu:
  - Lateral
---
<section>
  <p>
    Creamos la base de datos de "keystone" y otorgamos los permisos necesarios.
    <pre>
      root@nova-controller:~# mysql -u root -p
      MariaDB [(none)]> CREATE DATABASE keystone;
      MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
      MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
    </pre>

    Generamos de forma aleatoria un token para realizar la configuración inicial.
    <pre>
      root@nova-controller:~# openssl rand -hex 10
    </pre>

    Deshabilitamos keystone para que no se inicie automáticamente tras la instalación.
    <pre>
      root@nova-controller:~# echo "manual" > /etc/init/keystone.override
    </pre>

    Instalamos los paquetes necesarios.
    <pre>
      root@keystone:~# apt install keystone apache2 libapache2-mod-wsgi
    </pre>

    Editamos el fichero <b>/etc/keystone/keystone.conf</b> y añadimos las siguientes líneas.
    <pre>
      root@keystone:~# emacs /etc/keystone/keystone.conf
      [DEFAULT]
      admin_token=ADMIN_TOKEN
      ...
      [database]
      ...
      connection=mysql+pymysql://keystone:KEYSTONE_DBPASS@NOVA_CONTROLLER_INTERNAL_IP/keystone
      ...
      [token]
      ...
      provider=fernet
    </pre>
    <blockquote>
      ADMIN_TOKEN es el token generado anteriormente de forma aleatoria.
    </blockquote>

    Poblamos la base de datos de Keystone.
    <pre>
      root@keystone:~# su -s /bin/sh -c "keystone-manage db_sync" keystone
    </pre>

    Creamos la primera Fernet key.
    <pre>
      root@keystone:~# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
    </pre>

    Editamos el fichero de configuración de apache para que el <b>ServerName</b> haga referencia a la dirección IP de la interfaz "eth1" del lxc KEYSTONE.
    <pre>
      root@keystone:~# emacs /etc/apache2/apache2.conf
      ServerName KEYSTONE_INTERNAL_IP
    </pre>

    Creamos un nuevo sitio web para keystone.
    <pre>
      root@keystone:~# emacs /etc/apache2/sites-available/wsgi-keystone.conf
      Listen 5000
      Listen 35357

      <VirtualHost *:5000>
          WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
	  WSGIProcessGroup keystone-public
	  WSGIScriptAlias / /usr/bin/keystone-wsgi-public
	  WSGIApplicationGroup %{GLOBAL}
	  WSGIPassAuthorization On
	  ErrorLogFormat "%{cu}t %M"
	  ErrorLog /var/log/apache2/keystone.log
	  CustomLog /var/log/apache2/keystone_access.log combined

	  <Directory /usr/bin>
	      Require all granted
          </Directory>
      </VirtualHost>

      <VirtualHost *:35357>
          WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
	  WSGIProcessGroup keystone-admin
	  WSGIScriptAlias / /usr/bin/keystone-wsgi-admin
	  WSGIApplicationGroup %{GLOBAL}
	  WSGIPassAuthorization On
	  ErrorLogFormat "%{cu}t %M"
	  ErrorLog /var/log/apache2/keystone.log
	  CustomLog /var/log/apache2/keystone_access.log combined

          <Directory /usr/bin>
              Require all granted
          </Directory>
      </VirtualHost>
    </pre>

    Habilitamos el nuevo sitio web.
    <pre>
      root@keystone:~# a2ensite wsgi-keystone.conf
    </pre>

    Reiniciamos Apache.
    <pre>
      root@keystone:~# service apache2 restart
    </pre>

    Eliminamos la base de datos SQLite.
    <pre>
      root@keystone:~# rm -f /var/lib/keystone/keystone.db
    </pre>

    Declaramos las variables de entorno necesarias para crear las entidades de los servicios de nuestro OpenStack.
    <pre>
      root@keystone:~# export OS_TOKEN=1783cf34785391bfdf33
      root@keystone:~# export OS_URL=http://KEYSTONE_IP:35357/v3
      root@keystone:~# export OS_IDENTITY_API_VERSION=3
    </pre>

    Creamos la entidad del servicio de identidad (Keystone).
    <pre>
      root@keystone:~# openstack service create --name keystone --description "OpenStack Identity" identity
      +-------------+----------------------------------+
      | Field       | Value                            |
      +-------------+----------------------------------+
      | description | OpenStack Identity               |
      | enabled     | True                             |
      | id          | 7f1f794e61fe4443999b4a426c85381d |
      | name        | keystone                         |
      | type        | identity                         |
      +-------------+----------------------------------+
    </pre>

    Creamos los endpoints de Keystone.
    <pre>
      root@keystone:~# openstack endpoint create --region RegionOne identity public http://KEYSTONE_INTERNAL_IP:5000/v3
      root@keystone:~# openstack endpoint create --region RegionOne identity internal http://KEYSTONE_INTERNAL_IP:5000/v3
      root@keystone:~# openstack endpoint create --region RegionOne identity admin http://KEYSTONE_INTERNAL_IP:35357/v3
    </pre>

    Creamos un nuevo dominio en OpenStack.
    <pre>
      root@keystone:~# openstack domain create --description "Default Domain" default
    </pre>

    Creamos el proyecto, el usuario y el rol para el usuario "admin".
    <pre>
      root@keystone:~# openstack project create --domain default --description "Admin Project" admin
      root@keystone:~# openstack user create --domain default --password-prompt admin
      root@keystone:~# openstack role create admin
    </pre>
    
    Añadimos el rol admin a su correspondiente usuario y proyecto.
    <pre>
      root@keystone:~# openstack role add --project admin --user admin admin
    </pre>

    Creamos el proyecto "service" que alojará los usuarios creados para cada servicio añadido a OpenStack.
    <pre>
      root@keystone:~# openstack project create --domain default --description "Service Project" service
    </pre>
    
    Creamos también un usuario sin permisos de administración.
    <pre>
      root@keystone:~# openstack project create --domain default --description "Diego Project" diego
      root@keystone:~# openstack user create --domain default --password-prompt diego
    </pre>

    Creamos un rol "user".
    <pre>
      root@keystone:~# openstack role create user
      root@keystone:~# openstack role add --project diego --user diego user
    </pre>

    Deshabilitamos el mecanismo de token de autenticación temporal. Editamos el fichero <b>/etc/keystone/keystone-paste.ini</b> y eliminamos el admin_token_auth en las secciones "[pipeline:public_api]", "[pipeline:admin_api]" y "[pipeline:api_v3]".
    Borramos de memoria las variables OS_TOKEN y OS_URL.
    <pre>
      root@keystone:~# unset OS_TOKEN OS_URL
    </pre>

    Solicitamos un token de autenticación como usuario "admin".
    <pre>
      root@keystone:~# openstack --os-auth-url http://KEYSTONE_INTERNAL_IP:35357/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name admin --os-username admin token issue
    </pre>
    
    Solicitamos un token de autenticación como usuario "diego".
    <pre>
      root@keystone:~# openstack --os-auth-url http://KEYSTONE_INTERNAL_IP:5000/v3 --os-project-domain-name default --os-user-domain-name default --os-project-name diego --os-username diego token issue
    </pre>

    Creamos los "OpenStack client scripts".
    <pre>
      root@CLIENT:~$ emacs admin-openrc
      #!/bin/bash
      export OS_PROJECT_DOMAIN_NAME=default
      export OS_USER_DOMAIN_NAME=default
      export OS_PROJECT_NAME=admin
      export OS_USERNAME=admin
      export OS_PASSWORD=ADMIN_PASS
      export OS_AUTH_URL=http://KEYSTONE_INTERNAL_IP:35357/v3
      export OS_IDENTITY_API_VERSION=3
      export OS_IMAGE_API_VERSION=2
    </pre>
    <pre>
      root@CLIENT:~$ emacs diego-openrc
      #!/bin/bash
      export OS_PROJECT_DOMAIN_NAME=default
      export OS_USER_DOMAIN_NAME=default
      export OS_PROJECT_NAME=diego
      export OS_USERNAME=diego
      export OS_PASSWORD=USER_PASS
      export OS_AUTH_URL=http://KEYSTONE_IP:5000/v3
      export OS_IDENTITY_API_VERSION=3
      export OS_IMAGE_API_VERSION=2
    </pre>
  </p>
