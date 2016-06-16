---
layout: blog
title: Documentación
navbar:
  - Documentación
menu:
  - Lateral
---
<section>
  <h2>Entorno</h2>
  <img src="images/os02.png"/>
  <p>
    <b>Red interna:</b> Red utilizada para la comunicación entre los distintos servicios que componen el entorno de nuestro OpenStack. Esta red debe estar aislada de la red externa para garantizar que la comunicación no es interceptada desde el exterior.
    <br/>
    <b>Red externa:</b> Red que conecta a cada contenedor con el exterior. Usadas para conectar remotamente a cada servicio.
  </p>
  <h3>Contenedores</h3>
  <p>
    <b>Nodo Controlador:</b><br/>
    <pre>
      root@CONTROLLER:~# lxc-ls -f
      NAME            STATE   AUTOSTART GROUPS IPV4                   IPV6
      glance          RUNNING 1         -      10.0.200.12, 10.0.3.12 -
      horizon         RUNNING 1         -      10.0.200.13, 10.0.3.13 -
      keystone        RUNNING 1         -      10.0.200.14, 10.0.3.14 -
      nova-controller RUNNING 1         -      10.0.200.16, 10.0.3.16 -   
    </pre>
    <blockquote>
      Los servicios neutron
    </blockquote>
    <b>Nodo de Computo:</b><br/>
    <pre>
      root@COMPUTE:~# lxc-ls -f
      NAME          STATE   AUTOSTART GROUPS IPV4                                  IPV6
      nova_computer RUNNING 1         -      10.0.200.18, 10.0.3.18, 192.168.122.1 -   
    </pre>
    <p>
      <b>Sistema Operativo:</b> Ubuntu 14.04 Trusty<br/>
      <b>OpenStack:</b> Mitaka<br/>
    </p>
  </p>
