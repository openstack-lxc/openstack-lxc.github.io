---
layout: blog
title: Nova Computer - En el nodo de computo
navbar:
  - Documentación
menu:
  - Lateral
---
<section>
  <p>
    En el contenedor <b>nova-computer</b>, instalamos el paquete <b>nova-compute</b>.
    <pre>
      root@nova-computer:~# apt install nova-compute
    </pre>

    Editamos el fichero de configuración <b>/etc/nova/nova.conf</b> y añadimos las siguientes líneas.
    <pre>
      root@nova-computer:~# emacs /etc/nova/nova.conf
      [DEFAULT]
      dhcpbridge_flagfile=/etc/nova/nova.conf
      dhcpbridge=/usr/bin/nova-dhcpbridge
      state_path=/var/lib/nova
      lock_path=/var/lock/nova
      force_dhcp_release=True
      libvirt_use_virtio_for_bridges=True
      ec2_private_dns_show_ip=True
      api_paste_config=/etc/nova/api-paste.ini
      enabled_apis=ec2,osapi_compute,metadata
      rpc_backend=rabbit
      auth_strategy=keystone
      use_neutron=True
      firewall_driver=nova.virt.firewall.NoopFirewallDriver

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
     enabled=True
     vncserver_listen=0.0.0.0
     vncserver_proxyclient_address=NOVA_COMPUTER_INTERNAL_IP
     novncproxy_base_url=http://NOVA_CONTROLLER_INTERNAL_IP:6080/vnc_auto.html

     [glance]
     api_servers=http://GLANCE_INTERNAL_IP:9292

     [oslo_concurrency]
     lock_path=/var/lib/nova/tmp
    </pre>

    Reiniciamos el servicio nova-compute.
    <pre>
      root@nova-computer:~# service nova-compute restart
    </pre>
  </p>
