---
layout: blog
title: Glance
navbar:
  - Documentación
menu:
  - Lateral
---
<section>
  <p>
    Creamos la base de datos de "glance" y otorgamos los permisos necesarios.
    <pre>
      root@nova-controller:~# mysql -u root -p
      MariaDB [(none)]> CREATE DATABASE glance;
      MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
      MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
    </pre>

    Cargamos las credenciales del usuario Admin.
    <pre>
      root@client:~$ source admin-openrc      
    </pre>

    Creamos el usuario "glance", lo añadimos al rol "admin" y al projecto "service".
    <pre>
      root@client:~$ openstack user create --domain default --password-prompt glance
      root@client:~$ openstack role add --project service --user glance admin
    </pre>
    
    Creamos la entidad de servicio para <b>glance</b>.
    <pre>
      root@client:~$ openstack service create --name glance --description "OpenStack Image" image
    </pre>

    Creamos los endpoints para la API de <b>glance</b>.
    <pre>
      root@client:~$ openstack endpoint create --region RegionOne image public http://GLANCE_IP:9292
      root@client:~$ openstack endpoint create --region RegionOne image internal http://GLANCE_IP:9292
      root@client:~$ openstack endpoint create --region RegionOne image admin http://GLANCE_IP:9292
    </pre>

    Accedemos al contenedor de <b>glance</b> e instalamos los paquetes necesarios.
    <pre>
      root@glance:~# apt install glance python-memcache
    </pre>
    
    Editamos el fichero <b>/etc/glance/glance-api.conf</b> y añadimos las siguientes líneas.
    <pre>
      root@glance:~# emacs /etc/glance/glance-api.conf
      ...
      [database]
      connection = mysql+pymysql://glance:GLANCE_DBPASS@NOVA_CONTROLLER_INTERNAL_IP/glance
      ...
      keystone_authtoken]
      ...
      auth_uri = http://KEYSTONE_INTERNAL_IP:5000
      auth_url = http://KEYSTONE_INTERNAL_IP:35357
      memcached_servers = NOVA_CONTROLLER_INTERNAL_IP:11211
      auth_type = password
      project_domain_name = default
      user_domain_name = default
      project_name = service
      username = glance
      password = GLANCE_PASS
      ...
      [paste_deploy]
      ...
      flavor = keystone
      ...
      [glance_store]
      stores = file,http
      default_store = file
      filesystem_store_datadir = /var/lib/glance/images/
    </pre>

    Editamos el fichero <b>/etc/glance/glance-registry.conf</b> y añadimos las siguientes líneas.
    <pre>
      root@glance:~# emacs /etc/glance/glance-registry.conf
      ...
      [database]
      ...
      connection = mysql+pymysql://glance:GLANCE_DBPASS@NOVA_CONTROLLER_INTERNAL_IP/glance
      ...
      [keystone_authtoken]
      ...
      auth_uri = http://KEYSTONE_INTERNAL_IP:5000
      auth_url = http://KEYSTONE_INTERNAL_IP:35357
      memcached_servers = NOVA_CONTROLLER_INTERNAL_IP:11211
      auth_type = password
      project_domain_name = default
      user_domain_name = default
      project_name = service
      username = glance
      password = GLANCE_PASS
      ...
      [paste_deploy]
      flavor = keystone
    </pre>

    Poblamos la base de datos de Glance.
    <pre>
      root@glance:~# su -s /bin/sh -c "glance-manage db_sync" glance
    </pre>

    Reiniciamos los servicios de glance.
    <pre>
      root@glance:~# service glance-registry restart
      root@glance:~# service glance-api restart
    </pre>
  </p>
