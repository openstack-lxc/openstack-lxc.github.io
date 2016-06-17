---
layout: blog
title: Nova Computer - En el nodo controlador
navbar:
  - Documentación
menu:
  - Lateral
---
<section>
  <p>
    Creamos las bases de datos necesarias para "nova" y otorgamos los permisos necesarios.
    <pre>
      root@nova-controller:~# mysql -u root -p
      MariaDB [(none)]> CREATE DATABASE nova_api;
      MariaDB [(none)]> CREATE DATABASE nova;
      MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
      MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
      MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
      MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
    </pre>

    Cargamos las credenciales del usuario Admin.
    <pre>
      root@client:~$ source admin-openrc
    </pre>

    Creamos el usuario "nova", lo añadimos al rol "admin" y al projecto "service".
    <pre>
      root@client:~$ openstack user create --domain default --password-prompt nova
      root@client:~$ openstack role add --project service --user nova admin
    </pre>

    Creamos la entidad de servicio para <b>nova-computer</b>.
    <pre>
      root@os-client:~$ openstack service create --name nova --description "OpenStack Compute" compute
    </pre>

    Creamos los endpoints para la API de <b>nova-compute</b>.
    <pre>
      root@os-client:~$ openstack endpoint create --region RegionOne compute public http://NOVA_CONTROLLER_IP:8774/v2.1/%\(tenant_id\)s
      root@os-client:~$ openstack endpoint create --region RegionOne compute internal http://NOVA_CONTROLLER_IP:8774/v2.1/%\(tenant_id\)s
      root@os-client:~$ openstack endpoint create --region RegionOne compute admin http://NOVA_CONTROLLER_IP:8774/v2.1/%\(tenant_id\)s
    </pre>

    Instalamos los paquetes necesarios en el nodo controlador, <b>nova-controller</b>.
    <pre>
      root@nova-controller:~# apt install -y nova-api nova-conductor nova-consoleauth nova-novncproxy nova-scheduler
    </pre>

    Editamos en <b>nova_controller</b> el fichero de configuración <b>/etc/nova/nova.conf</b>.
    <pre>
      root@nova-controller:~# emacs /etc/nova/nova.conf
      [DEFAULT]
      dhcpbridge_flagfile=/etc/nova/nova.conf
      dhcpbridge=/usr/bin/nova-dhcpbridge
      state_path=/var/lib/nova
      lock_path=/var/lock/nova
      force_dhcp_release=True
      libvirt_use_virtio_for_bridges=True
      ec2_private_dns_show_ip=True
      api_paste_config=/etc/nova/api-paste.ini
      enabled_apis=osapi_compute,metadata
      rpc_backend=rabbit
      auth_strategy=keystone
      use_neutron=True
      firewall_driver=nova.virt.firewall.NoopFirewallDriver

      [api_database]
      connection=mysql+pymysql://nova:NOVA_DB_PASS@NOVA_CONTROLLER_INTERNAL_IP/nova_api

      [database]
      connection=mysql+pymysql://nova:NOVA_DB_PASS@NOVA_CONTROLLER_INTERNAL_IP/nova

      [oslo_messaging_rabbit]
      rabbit_host=NOVA_CONTROLLER_INTERNAL_IP
      rabbit_userid=openstack
      rabbit_password=RABBIT_PASS

      [keystone_authtoken]
      auth_uri=http://KEYSTONE_INTERNAL_IP:5000
      auth_url=http://KEYSTONE_INTERNAL_IP:35357
      memcached_servers=KEYSTONE_INTERNAL_IP:11211
      auth_type=password
      project_domain_name=default
      user_domain_name=default
      project_name=service
      username=nova
      password=NOVA_PASS

      [vnc]
      vncserver_listen=NOVA_CONTROLLER_INTERNAL_IP
      vncserver_proxyclient_address=NOVA_CONTROLLER_INTERNAL_IP

      [glance]
      api_servers=http://GLANCE_INTERNAL_IP:9292

      [oslo_concurrency]
      lock_path=/var/lib/nova/tmp
    </pre>

    Reiniciamos los servicios.
    <pre>
      root@nova-controller:~# service nova-api restart
      root@nova-controller:~# service nova-consoleauth restart
      root@nova-controller:~# service nova-scheduler restart
      root@nova-controller:~# service nova-conductor restart
      root@nova-controller:~# service nova-novncproxy restart
    </pre>
  </p>
