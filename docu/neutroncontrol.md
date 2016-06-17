---
layout: blog
title: Neutron - En el nodo controlador
navbar:
  - Documentación
menu:
  - Lateral
---
<section>
  <p>
    Neutron, es junto a Cinder los dos servicios que no aislaremos en contenedores, ya que esto supondría tener que otorgar permisos a dichos contenedores que harían que el aislamiento de estos fuera inexistente. Por lo tanto, Neutro y Cinder serán instalados en el Host <b>CONTROLLER</b>.
  </p>
  <p>
    En <b>nova-controller</b>, creamos la base de datos y el usuario para "neutron", y otorgamos los permisos necesarios.
    <pre>
      root@nova-controller:~# mysql -u root -p
      MariaDB [(none)]> CREATE DATABASE neutron;
      MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
      MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'NEUTRON_DBPASS';
    </pre>

    Cargamos las credenciales del usuario Admin.
    <pre>
      user@client:~$ source admin-openrc
    </pre>

    Creamos el usuario "neutron", lo añadimos al rol "admin" y al projecto "service".
    <pre>
      user@client:~$ openstack user create --domain default --password-prompt neutron
      user@client:~$ openstack role add --project service --user neutron admin
    </pre>

    Creamos la entidad de servicio para *neutron*.
    <pre>
      user@client:~$ openstack service create --name neutron --description "OpenStack Networking" network
    </pre>

    Creamos los endpoints para la API de <b>neutron</b>.
    <pre>
      user@client:~$ openstack endpoint create --region RegionOne network public http://NEUTRON_INTERNAL_IP:9696
      user@client:~$ openstack endpoint create --region RegionOne network internal http://NEUTRON_INTERNAL_IP:9696
      user@client:~$ openstack endpoint create --region RegionOne network admin http://NEUTRON_INTERNAL_IP:9696
    </pre>

    En el Host <b>CONTROLLER</b>, instalamos los siguientes paquetes:
    <pre>
      root@CONTROLLER:~# apt install neutron-server neutron-plugin-ml2 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent python-memcache
    </pre>

    Editamos el fichero <b>/etc/neutron/neutron.conf</b> y añadimos las siguiente líneas.
    <pre>
      root@CONTROLLER:~# emacs /etc/neutron/neutron.conf
      [DEFAULT]
      auth_strategy=keystone
      core_plugin=ml2
      service_plugins = router
      allow_overlapping_ips=True
      notify_nova_on_port_status_changes=True
      notify_nova_on_port_data_changes=True
      rpc_backend=rabbit

      [agent]
      root_helper=sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf

      [database]
      connection=mysql+pymysql://neutron:NEUTRON_DBPASS@NOVA_CONTROLLER_INTERNAL_IP/neutron

      [keystone_authtoken]
      auth_uri=http://KEYSTONE_INTERNAL_IP:5000
      auth_url=http://KEYSTONE_INTERNAL_IP:35357
      memcached_servers = NOVA_CONTROLLER_INTERNAL_IP:11211
      auth_type=password
      project_domain_name=default
      user_domain_name=default
      region_name=RegionOne
      project_name=service
      username=nova
      password=NOVA_PASS

      [nova]
      auth_url=http://KEYSTONE_INTERNAL_IP:35357
      auth_type=password
      project_domain_name=default
      user_domain_name=default
      project_name=service
      username=nova
      password=NOVA_PASS

      [oslo_messaging_rabbit]
      rabbit_host=NOVA_CONTROLLER_INTERNAL_IP
      rabbit_userid=openstack
      rabbit_password=RABBIT_PASS
    </pre>

    El plug-in ML2 es el encargado de crear la infraestructura de red virtual para las instancias, usando Linux bridge. Para configurar correctamente este plug-in, debemos editar el fichero <b>/etc/neutron/plugins/ml2/ml2_conf.ini</b>.
    <pre>
      root@CONTROLLER:~# emacs /etc/neutron/plugins/ml2/ml2_conf.ini
      [DEFAULT]
      [ml2]
      type_drivers = flat,vxlan
      tenant_network_types = vxlan
      mechanism_drivers = linuxbridge,l2population
      extension_drivers = port_security

      [ml2_type_flat]
      flat_networks = provider
      
      [ml2_type_vxlan]
      vni_ranges = 1:1000

      [securitygroup]
      enable_ipset = True
    </pre>

    Configuramos el agente de Linux bridge para el plug-in ml2.
    <pre>
      root@CONTROLLER:~# emacs /etc/neutron/plugins/ml2/linuxbridge_agent.ini
      [DEFAULT]
      [linux_bridge]
      physical_interface_mappings = provider:br0

      [securitygroup]
      enable_security_group = True
      firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

      [vxlan]
      enable_vxlan = true
      local_ip = NEUTRON_INTERNAL_IP
      l2_population = True
    </pre>

    Para poder hacer uso de uso de <b>routing</b> y <b>NAT</b> en nuestras redes virtuales, debemos configurar el agente <b>layer-3</b>.
    <pre>
      root@CONTROLLER:~# emacs /etc/neutron/l3_agent.ini
      [DEFAULT]
      interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
      external_network_bridge =
    </pre>

    Nuestras redes virtuales deben contar también con un servidor DHCP.
    <pre>
      root@CONTROLLER:~# emacs /etc/neutron/dhcp_agent.ini
      [DEFAULT]
      interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
      dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
      enable_isolated_metadata = True
    </pre>

    Editamos el fichero <b>/etc/neutron/metadata_agent.ini</b> para que apunte al servidor de metadatos en nova-controller.
    <pre>
      root@CONTROLLER:~# emacs /etc/neutron/metadata_agent.ini
      [DEFAULT]
      nova_metadata_ip = NOVA_CONTROLLER_INTERNAL_IP
      metadata_proxy_shared_secret = METADATA_SECRET
    </pre>

    Editamos el fichero <b>/etc/nova/nova.conf</b> para añadir los nuevos cambios realizados en el contenedor <b>nova-controller</b>.
    <pre>
      root@nova-controller:~# emacs /etc/nova/nova.conf
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
      password = NEUTRON_PASS
      service_metadata_proxy = True
      metadata_proxy_shared_secret = METADATA_SECRET
    </pre>

    Poblamos la base de datos.
    <pre>
      su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
    </pre>

    Reiniciamos los servicios para actualizar los cambios.
    <pre>
      root@nova-controller:~# service nova-api restart
    </pre>
    <pre>
      root@CONTROLLER:~# service neutron-server restart
      root@CONTROLLER:~# service neutron-linuxbridge-agent restart
      root@CONTROLLER:~# service neutron-dhcp-agent restart
      root@CONTROLLER:~# service neutron-metadata-agent restart
      root@CONTROLLER:~# service neutron-l3-agent restart
    </pre>
  </p>
