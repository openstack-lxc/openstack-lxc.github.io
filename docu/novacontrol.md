---
layout: blog
title: Nova Controller
navbar:
  - Documentación
menu:
  - Lateral
---
<section>
  <p>
    Accedemos al contenedor <b>nova-controller</b>.
    <pre>
      root@OSNODE:~# lxc-attach -n nova-controller
    </pre>
    Configuramos MariaDB en Nova_controller para almacenar los datos de nuestro OpenStack. Para ello, instalamos los siguientes paquetes:
    <pre>
      root@nova-controller:~# apt install mariadb-server python-pymysql
    </pre>
    Creamos el fichero "/etc/mysql/conf.d/openstack.cnf".
    <pre>
      root@nova-controller:~# emacs /etc/mysql/conf.d/openstack.cnf
      [mysqld]
      bind-address = NOVA_CONTROLLER_INTERNAL_IP
      default-storage-engine = innodb
      innodb_file_per_table
      collation-server = utf8_general_ci
      character-set-server = utf8
    </pre>
    <blockquote>
      NOVA_CONTROLLER_INTERNAL_IP = 10.0.3.16
    </blockquote>
    Reiniciamos el servicio.
    <pre>
      root@nova-controller:~# service mysql restart
    </pre>
  </p>
  <p>
    Instalamos RabbitMQ.
    <pre>
      root@nova-controller:~# apt install rabbitmq-server
    </pre>
    Añadimos el usuario "openstack".
    <pre>
      root@nova-controller:~# rabbitmqctl add_user openstack RABBIT_USER_PASS
    </pre>
    Damos permiso de configuración, escritura y lectura al usuario "openstack".
    <pre>
      root@nova-controller:~# rabbitmqctl set_permissions openstack ".*" ".*" ".*"
    </pre>
  </p>
  <p>
    Instalamos Memcache y el cliente desarrollado en python.
    <pre>
      root@nova-controller:~# apt install memcached python-memcache
    </pre>
    Configuramos memcached para que escuche las peticiones que lleguen desde los demás equipos de la red local.
    <pre>
      root@nova-controller:~# emacs /etc/memcached.conf
      -l NOVA_CONTROLLER_INTERNAL_IP
    </pre>
    Reiniciamos el servicio.
    <pre>
      root@nova-controller:~# service memcached restart
    </pre>
  </p>
