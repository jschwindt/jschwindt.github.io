---
layout: post
title: "Redis Cluster con Ruby"
tags: [ruby, redis]
---
## Intro

En [Comenta.Tv](http://comenta.tv) procesamos gran cantidad de tweets, que obtenemos
a partir de la [API de streaming](http://redis.io/) de Twitter, y generamos estadísticas
y reportes en tiempo real.

Para lograr escalar a cientos de mensajes por segundo y para poder generar los reportes
con tiempos de respuesta rápidos, nos valemos principalmente de [Redis](http://redis.io/),
que demostró ser una herramienta ideal para logar nuestra meta.

El "problema" de Redis (y también su virtud) es que toda la base de datos tiene que caber
en la memoria RAM, y rápidamente esto se convierte en un recurso escaso. La solución ideal
sería disponer ya de [Redis Cluster](http://redis.io/topics/cluster-spec), pero como todavía
no está disponible hubo que encarar una solución más casera para particionar el contenido
en distintos servidores.

## Redis::Distributed

Según [la documentación](http://redis.io/topics/partitioning) de Redis, hay distintas
opciones para particionar los datos en múltiples servidores y una de las opciones es
utilizar directamente el cliente de [Redis en Ruby](https://github.com/redis/redis-rb)
utilizando la opción "distributed".

La documentación para utilizar `Redis::Distributed` es casi nula y encontré pocos ejemplos
y demasiado básicos como para tomarlos en serio. Luego de cometer algunos errores que me
llevaron a andar moviendo millones de claves de un servidor a otro, decidí escribir este
post para tratar de evitar que otros pasen por lo mismo.

### ¿Cómo funciona?

Normalmente para utilizar Redis desde Ruby con un sólo servidor hacemos algo así:

```ruby
client = Redis.new('redis://127.0.0.1:6379/0')

client.set "clave1", "Hello!"        # "OK"
client.get "clave1"                  # "Hello!"
client.del "clave1"                  # 1

```

Cuando utilizamos `Redis::Distributed` el uso es similar, pero el cliente se inicializa
con un array de servidores Redis en lugar de un único servidor:

```ruby
REDIS_ARRAY = [ 'redis://127.0.0.1:6379/0', 'redis://127.0.0.1:6380/0' ]

client = Redis::Distributed.new(REDIS_ARRAY)
client.set "clave1", "Hello!"        # "OK"
client.get "clave1"                  # "Hello!"
```

Como vemos ahora tenemos múltiples server, pero el funcionamiento del los comandos es
similar. Ahora el servidor de destino se elige en base al valor de la clave:

```ruby
client.node_for "clave1"             # <Redis client v3.0.4 for redis://127.0.0.1:6379/0>
client.node_for "clave2"             # <Redis client v3.0.4 for redis://127.0.0.1:6380/0>
```

### Limitaciones

#### Múltiples claves

El hecho de que cada clave se encuentre en un servidor distinto impone algunas limitaciones,
por ejemplo: todas las operaciones que impliquen múltiples claves no son posibles:

```ruby
client = Redis::Distributed.new(REDIS_ARRAY)

client.mset("clave1", "val1", "clave2", "val2")
  >> Redis::Distributed::CannotDistribute: Redis::Distributed::CannotDistribute
```

Esto no significó un problema para nosotros porque no utilizamos ninguna de dichas
operaciones ya que todo lo realizamos en base a claves independientes.

#### Agregar servidores

Una vez que uno tiene el cluster de servidores funcionando y con contenido, no es posible
agregar un nuevo servidor al array porque eso alteraría automáticamente la distribución
de claves y las ya existentes quedaría inaccesibles.

Entonces, ¿cómo hacer cuando nos volvemos a quedar sin memoria? La primera opción sería
agregar memoria a los servidores actuales, pero esto no siempre es posible.

La solución sugerida es que en el armado del cluster tengamos la previsión de tener
muchos servidores Redis corriendo, aunque inicialmente todos los procesos pueden estar
en el mismo servidor físico y a medida que necesitamos, vamos migrando el contenido
a nuevos servidores físicos. La cantidad de servidores no debemos cambiarla nunca y
no hay que tener miedo de tener muchos servidores Redis corriendo en el mismo server
físico ya que la utilización de memoria y CPU es muy baja.

### Armado del cluster

#### Identificar los servidores por ID

En los ejemplos que había visto con `Redis::Distributed` la inicialización se realizaba
siempre especificando un array de urls de redis, tal como se ven en los ejemplos
anteriores, pero para mi sorpresa la distribución de claves no se realiza en base a
la posición del servidor dentro del array sino en base a su ID, que por defecto es la
url del servidor. Por lo tanto, si cambiamos la IP o el puerto de un servidor redis, ¡la
distribución de claves cambia!

La solución es especificar el array de servidores de la siguiente forma:

```ruby
REDIS_ARRAY = [
  { id:  'redis_0',
    url: "redis://127.0.0.1:6379/0",
  },
  { id:  'redis_1',
    url: "redis://127.0.0.1:6380/0",
  },
  #
  # more servers
  #
  { id:  'redis_n',
    url: "redis://127.0.0.1:63xx/0",
  },
]
```

De esta forma, cuando agregamos un nuevo servidor físico para aumentar
la capacidad del cluster, lo único que tenemos que hacer es [replicar la base
de datos de Redis](http://redis.io/topics/replication) y luego cambiar sólo
la url correspondiente en el REDIS_ARRAY.

#### Script de arranque para Ubuntu 12.04

Para simplificar el arranque y detención de los múltiples servidores Redis que
corren en el mismo servidor físico, modifiqué el script alojado en /etc/init.d/
para agregar fácilmente múltiples servidores en distintos puertos. En el siguiente
[gist]((https://gist.github.com/jschwindt/6312036#file-redis-__port__)) se puede ver el archivo
[`/etc/init.d/redis-__port__`](https://gist.github.com/jschwindt/6312036#file-redis-__port__)
que sirve de base para crear los servidores.

Una vez que tenemos dicho archivo en `/etc/init.d/` y con los permisos de ejecución correspondientes,
lo único que hace falta hacer es crear links simbólicos cuyo nombre tiene que contener el port
en el que queremos que el servidor Redis esté escuchando:

```bash
cd /etc/init.d/

ln -s redis-__port__ redis-6380
ln -s redis-__port__ redis-6381
ln -s redis-__port__ redis-6382
ln -s redis-__port__ redis-6383

update-rc.d redis-6380 defaults
update-rc.d redis-6381 defaults
update-rc.d redis-6382 defaults
update-rc.d redis-6383 defaults

```

También es necesario tener el archivo [`/etc/redis/_common.conf`](https://gist.github.com/jschwindt/6312036#file-_common-conf)
que contiene toda la configuración común a todas las instancias. Opcionalmente también se
pueden especificar configuraciones particulares a cada instancia creando el archivo correspondiente
en [`/etc/redis/_\[port\]`](https://gist.github.com/jschwindt/6312036#file-_6380-conf).

Al arrancar un servidor, por ejemplo con:

```bash
/etc/init.d/redis-6380 start
```

el script se encarga de crear la configuración correspondiente en `/etc/redis/redis-6380.conf` y luego
arrancar la instancia con dicha configuración, que como ejemplo va a tener el siguiente contenido:

```
port 6380
pidfile /var/run/redis/redis-6380.pid
logfile /var/log/redis/redis-6380.log
dbfilename dump-6380.rdb
include /etc/redis/_common.conf
```

Si además detecta que existe el archivo `/etc/redis/_6380.conf`, la configuración tendrá además
incluido dicho archivo antes del `_common.conf`.

## Migración de claves

- TODO
