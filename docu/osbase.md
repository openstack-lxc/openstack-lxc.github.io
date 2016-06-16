---
layout: blog
title: OpenStack Pre-configuración
navbar:
  - Documentación
menu:
  - Lateral
---
<section>
  <p>
    En todos espacios de usuario donde vayan a instalarse paquetes de OpenStack, añadimos el repositorio de OpenStack y actualizamos los paquetes.
    <pre>
      root@OSNODE:~# apt update && apt install software-properties-common
      root@OSNODE:~# add-apt-repository cloud-archive:mitaka
      root@OSNODE:~# apt dist-upgrade
    </pre>
    Reiniciamos el sistema para que cargue el kernel en la versión 3.19.0.
    <pre>
      root@OSNODE:~# reboot
    </pre>
    Instalamos el CLI de OpenStack.
    <pre>
      root@OSNODE:~# apt install python-openstackclient
    </pre>
  </p>
