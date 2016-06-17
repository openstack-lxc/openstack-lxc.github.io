---
layout: blog
title: Horizon
navbar:
  - Documentación
menu:
  - Lateral
---
<section>
  <p>
    Instalamos horizon Mitaka en su respectivo contenedor.
    <pre>
      root@horizon:~# apt install openstack-dashboard python-memcache
    </pre>

    Editamos el fichero <b>/etc/openstack-dashboard/local_settings.py</b> y añadimos los siguientes cambios.
    <pre>
      ...
      OPENSTACK_HOST = "KEYSTONE_INTERNAL_IP"
      ...
      OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
      ...
      ALLOWED_HOSTS = ['*', ]
      ...
      SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
      CACHES = {
      	  'default': {
               'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
	       'LOCATION': 'NOVA_CONTROLLER_INTERNAL_IP:11211',
	  }
      }
      ...
      OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
      ...
      OPENSTACK_API_VERSIONS = {
          "identity": 3,
	  "image": 2,
	  "volume": 2,
      }
      ...
      OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "default"
      ...
      OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
      ...
      TIME_ZONE = "Europe/Madrid"
    </pre>

    Reiniciamos Apache.
    <pre>
      root@horizon:~# service apache2 reload
    </pre>
  </p>
