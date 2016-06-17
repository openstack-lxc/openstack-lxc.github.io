---
layout: blog
title: Neutron - En el nodo computador
navbar:
  - Documentación
menu:
  - Lateral
---
<section>
  <p>
    Instalamos el paquete <b>neutron-linuxbridge-agent</b>.
    <pre>
      root@nova-computer:~# apt install neutron-linuxbridge-agent
    </pre>

    Editamos el fichero <b>/etc/neutron/neutron.conf</b> y añadimos las siguientes líneas.
    <pre>
      root@nova-computer:~# emacs /etc/neutron/neutron.conf
      [DEFAULT]
      auth_strategy = keystone
      core_plugin = ml2
      rpc_backend = rabbit

      [agent]
      root_helper = sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf
      
      [database]
      connection = sqlite:////var/lib/neutron/neutron.sqlite

      [keystone_authtoken]
      auth_uri = http://KEYSTONE_INTERNAL_IP:5000
      auth_url = http://KEYSTONE_INTERNAL_IP:35357
      memcached_servers = NOVA_CONTROLLER_INTERNAL_IP:11211
      auth_type = password
      project_domain_name = default
      user_domain_name = default
      project_name = service
      username = neutron
      password = NEUTRON_DBPASS
      
      [oslo_messaging_rabbit]
      rabbit_host = NOVA_CONTROLLER_INTERNAL_IP
      rabbit_userid = openstack
      rabbit_password= RABBIT_PASS
    </pre>

    Editamos el fichero <b>/etc/neutron/plugins/ml2/linuxbridge_agent.ini</b>.
    <pre>
      root@nova-computer:~# emacs /etc/neutron/plugins/ml2/linuxbridge_agent.ini
      [DEFAULT]

      [linux_bridge]
      physical_interface_mappings = provider:br0

      [securitygroup]
      firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
      enable_security_group = True

      [vxlan]
      enable_vxlan = True
      local_ip = NOVA_COMPUTE_INTERNAL_IP
      l2_population = True
    </pre>

    Añadimos las siguientes líneas al fichero <b>/etc/nova/nova.conf</b>.
    <pre>
      root@nova-computer:~# emacs /etc/nova/nova.conf
      ...
      [neutron]
      url = http://HOST_CONTROLLER_INTERNAL_IP:9696
      auth_url = http://KEYSTONE_INTERNAL_IP:35357
      auth_type = password
      project_domain_name = default
      user_domain_name = default
      region_name = RegionOne
      project_name = service
      username = neutron
      password = NEUTRON_DBPASS
    </pre>

    Reiniciamos los servicios.
    <pre>
      root@nova-computer:~# service nova-compute restart
      root@nova-computer:~# service neutron-linuxbridge-agent restart
    </pre>
  </p>
  <p>
    Para crear las redes virtuales podemos seguir la <a href="http://docs.openstack.org/mitaka/install-guide-ubuntu/launch-instance-networks-provider.html">guía de OpenStack</a>.
  </p>
