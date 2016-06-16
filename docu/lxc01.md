---
layout: blog
title: LXC - Instalación
navbar:
  - Documentación
menu:
  - Lateral
---
<section>
  <p>
    Necesitamos instalar el paquete <b>lxc</b>.
    <pre>
      root@CONTROLLER:~# apt install -y lxc
      root@COMPUTE:~# apt install -y lxc
    </pre>
  </p>
  <p>
    En ambos hosts necesitamos crear el bridge al que conectaremos todas las instarfaces de los contenedores.
    <pre>
      root@CONTROLLER:~# emacs /etc/network/interfaces
      auto br0
      iface br0 inet dhcp
              bridge_ports eth0
	      bridge_fd 0
	      bridge_maxwait 0
	      up ip a add 10.0.3.15/24 dev br0
    </pre>
    Podemos establecer en LXC el uso por defecto del bridge creado.
    <pre>
      root@CONTROLLER:~# emacs /etc/lxc/default.conf
      lxc.network.link = br0
    </pre>
  </p>
  <p>
    Creamos los contenedores necesarios.
    <pre>
      root@CONTROLLER:~# lxc-create -t ubuntu -n keystone
      root@CONTROLLER:~# lxc-create -t ubuntu -n nova-controller
      root@CONTROLLER:~# lxc-create -t ubuntu -n horizon
      root@CONTROLLER:~# lxc-create -t ubuntu -n glance
    </pre>
    <pre>
      root@COMPUTE:~# lxc-create -t ubuntu -n nova-compute
    </pre>
  </p>
  <p>
    Editamos el fichero de configuración de cada contenedor y añadimos una nueva interfaz que usaremos para la comunicación interna de cada componente de OpenStack.
    <pre>
      root@CONTROLLER:~# emacs /var/lib/lxc/container/config
      ...
      # Network configuration
      lxc.network.type = veth
      lxc.network.link = br0
      lxc.network.flags = up
      lxc.network.hwaddr = IFACE_MAC
      lxc.network.name = eth0

      lxc.network.type = veth
      lxc.network.link = br0
      lxc.network.flags = up
      lxc.network.hwaddr = IFACE_MAC
      lxc.network.name = eth1
    </pre>
    Una vez conectados al contenedor, editamos la configuración de red de la interfaz interna (eth1).
    <pre>
      # lxc-attach -n container
      root@container:~# emacs /etc/network/interfaces
      auto eth1
      iface eth1 inet static
              address 10.0.3.X
	      netmask 255.255.255.0
	      network 10.0.3.0
	      broadcast 10.0.3.25
    </pre>
    Configuramos LXC para que inicie los contenedores automáticamente tras el arranque del sistema.
    <pre>
      root@CONTROLLER:~# emacs /var/lib/lxc/container/config
      lxc.start.auto = 1
      lxc.start.delay = 5
    </pre>
    <blockquote>
      lxc.start.delay: Tiempo de espera en segundos que transcurre desde que se ha terminado de iniciar un contenedor hasta que se inicia el siguiente.
    </blockquote>
  </p>
