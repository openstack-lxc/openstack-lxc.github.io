---
layout: blog
title: Cinder - Host CONTROLLER
navbar:
  - Documentación
menu:
  - Lateral
---
<section>
  <p>
    Instalamos lvm.
    <pre>
      root@CONTROLLER:~# apt install lvm2
    </pre>

    Creamos el grupo de volúmenes que usaremos para crear nuestros volúmenes en cinder.
    <pre>
      root@CONTROLLER:~# pvcreate /dev/sdb
      root@CONTROLLER:~# vgcreate cinder-volumes /dev/sdb
    </pre>

    Editamos la diguiente línea del fichero <b>/etc/lvm/lvm.conf</b>.
    <pre>
      root@CONTROLLER:~# /etc/lvm/lvm.conf
      devices {
      ...
      filter = [ "a/sdb/", "r/.*/"]
    </pre>

    Instalamos los paquetes necesario en el host CONTROLLER.
    <pre>
      root@CONTROLLER:~# apt install cinder-volume python-memcache
    </pre>

    Modificamos el fichero <b>/etc/cinder/cinder.conf</b>.
    <pre>
      root@CONTROLLER:~# /etc/cinder/cinder.conf
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
      my_ip = HOST_INTERNAL_IP
      enabled_backends = lvm
      default_volume_type = lvm
      glance_api_servers = http://GLANCE_INTERNAL_IP:9292
      
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
      password = CINDER_DBPASS

      [lvm]
      volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
      volume_group = cinder-volumes
      iscsi_protocol = iscsi
      iscsi_helper = tgtadm
      iscsi_iotype = fileio
      iscsi_ip_address = $my_ip

      [oslo_concurrency]
      lock_path = /var/lib/cinder/tmp
    </pre>

    Reiniciamos los servicios.
    <pre>
      root@CONTROLLER:~# service tgt restart
      root@CONTROLLER:~# service cinder-volume restart
    </pre>
  </p>
