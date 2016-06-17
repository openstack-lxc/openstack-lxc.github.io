---
layout: blog
title: Cinder - Nodo controlador
navbar:
  - Documentación
menu:
  - Lateral
---
<section>
  <p>
    En <b>nova-controller</b> creamos la base de datos para cinder.
    <pre>
      root@nova-controller:~# mysql -u root -p
      MariaDB [(none)]> CREATE DATABASE cinder;
      MariaDB [(none)]>	GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'CINDER_DBPASS';
      MariaDB [(none)]>	GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'CINDER_DBPASS';
    </pre>

    Creamos las credenciales para el servicio de cinder.
    <pre>
      user@client:~$ source admin-openrc
      user@client:~$ openstack user create --domain default --password-prompt cinder
      user@client:~$ openstack role add --project service --user cinder admin
    </pre>

    Creamos la entidad de los servicios <b>cinder</b> y <b>cinderv2</b>.
    <pre>
      user@client:~$ openstack service create --name cinder --description "OpenStack Block Storage" volume
      user@client:~$ openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
    </pre>

    Creamos los endpoints de cinder.
    <pre>
      user@client:~$ openstack endpoint create --region RegionOne volume public http://NOVA_CONTROLLER_INTERNAL_IP:8776/v1/%\(tenant_id\)s
      user@client:~$ openstack endpoint create --region RegionOne volume internal http://NOVA_CONTROLLER_INTERNAL_IP:8776/v1/%\(tenant_id\)s
      user@client:~$ openstack endpoint create --region RegionOne volume admin http://NOVA_CONTROLLER_INTERNAL_IP:8776/v1/%\(tenant_id\)s
      user@client:~$ openstack endpoint create --region RegionOne volumev2 public http://NOVA_CONTROLLER_INTERNAL_IP:8776/v2/%\(tenant_id\)s
      user@client:~$ openstack endpoint create --region RegionOne volumev2 internal http://NOVA_CONTROLLER_INTERNAL_IP:8776/v2/%\(tenant_id\)s
      user@client:~$ openstack endpoint create --region RegionOne volumev2 admin http://NOVA_CONTROLLER_INTERNAL_IP:8776/v2/%\(tenant_id\)s
    </pre>

    En <b>nova-controller</b> instalamos los siguientes paquetes.
    <pre>
      root@nova-controller:~# apt install cinder-api cinder-scheduler
    </pre>

    Editamos el fichero <b>/etc/cinder/cinder.conf</b>.
    <pre>
      root@nova-controller:~# emacs /etc/cinder/cinder.conf
      [DEFAULT]
      rootwrap_config = /etc/cinder/rootwrap.conf
      api_paste_confg = /etc/cinder/api-paste.ini
      iscsi_helper = tgtadm
      volume_name_template = volume-%s
      volume_group = cinder-volumes
      verbose = True
      auth_strategy = keystone
      state_path = /var/lib/cinder
      lock_path = /var/lock/cinder
      volumes_dir = /var/lib/cinder/volumes
      rpc_backend = rabbit
      auth_strategy = keystone
      my_ip = NOVA_CONTROLLER_INTERNAL_IP
      
      [database]
      connection = mysql+pymysql://cinder:CINDER_DBPASS@NOVA_CONTROLLER_INTERNAL_IP/cinder
      
      [oslo_messaging_rabbit]
      rabbit_host = NOVA_CONTROLLER_INTERNAL_IP
      rabbit_userid = openstack
      rabbit_password = RABBIT_PASS

      [keystone_authtoken]
      auth_uri = http://KEYSTONE_INTERNAL_IP:5000
      auth_url = http://KEYSTONE_INTERNAL_IP:35357
      memcached_servers = NOVA_CONTROLLER_INTERNAL_IP:11211
      auth_type = password
      project_domain_name = default
      user_domain_name = default
      project_name = service
      username = cinder
      password = CINDER_PASS
      [oslo_concurrency]
      lock_path = /var/lib/cinder/tmp
    </pre>

    Poblamos la base de datos.
    <pre>
      su -s /bin/sh -c "cinder-manage db sync" cinder
    </pre>

    Editamos el fichero <b>/etc/nova/nova.conf</b> y añadimos las siguientes líneas.
    <pre>
      root@nova-controller:~# emacs /etc/nova/nova.conf
      ...
      [cinder]
      os_region_name = RegionOne
    </pre>

    Reiniciamos los servicios.
    <pre>
      root@nova-controller:~# service nova-api restart
      root@nova-controller:~# service cinder-scheduler restart
      root@nova-controller:~# service cinder-api restart
    </pre>
  </p>
