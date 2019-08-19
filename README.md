# Instalación y configuración de LXC en Arch Linux

## Autores

- [Yoel Torres Vázquez - oneohthree](yoel.torres.v@gmail.com)
- [Ixen Rodríguez Pérez - kurosaki1976](ixenrp1976@gmail.com)

## Breve introducción a LXC

LXC es una interface de espacio de usuario para el soporte de contenedores sobre el kernel de Linux. Permite crear y administrar contenedores de sistemas o aplicaciones a través de herramientas sencillas y una potente API.

Los contenedores LXC frecuentemente son considerados como algo entre chroot y una máquina virtual. El objetivo fundamental de LXC es crear un entorno lo más cercano posible a una instalación estándar de Linux, pero sin la necesidad de un kernel independiente.

LXC actualmente se compone de varios componentes individuales: la biblioteca `liblxc`, varios bindings de lenguajes de programación para la API, tales como: Python 2 y 3, Lua, Go, Ruby y Haskell, un conjunto de heramientas estándar para controlar los contenedores, además de plantillas de contenedores de distribución.

LXC 2.0 y 3.0 son versiones LTS (soporte extendido), la primera tendrá soporte hasta junio de 2021 y la última hasta junio de 2023. La versión 3.2 está disponible en Arch Linux.

## Instalación de paquetes necesarios

```
pacman -S lxc lxcfs
```

El paquete `lxc` sugiere la instalación de `dnsmasq`, necesario para el correcto funcionamiento de la herramienta `lxc-net`, que permite configurar un bridge simple para los contenedores. Sin embargo, podemos prescindir de esta herramienta como se verá más adelante. Esta guía utiliza el modelo de red en el que se usa un bridge compartido entre el host y los guests.
Por otro lado, el paquete `lxcfs` permite definir los parámetros de límite de memoria RAM, swap y cpu que utilizarán los contenedores.

Un ejemplo de configuración con estos parámetros definidos, luciría así:

```
lxc.cgroup.cpuset.cpus = 1-2
lxc.cgroup.cpu.shares = 1024
lxc.cgroup.memory.limit_in_bytes = 768M
lxc.cgroup.memory.memsw.limit_in_bytes = 1G
```

Activar el servicio `lxcfs`.

```
systemctl start lxcfs
systemctl enable lxcfs

```

Para poder crear contenedores de Debian/Ubuntu, debemos instalar además:

```
pacman -S debootstrap debian-archive-keyring ubuntu-keyring
```

## Configuración de la red

El paquete de Arch Linux no incluye ninguna configuración de red por defecto para los contenedores. El archivo `/etc/lxc/default.conf` contiene solamente el valor `lxc.network.type = empty`.

A un contenedor típico se le puede asignar su propia interface de red física (**phys**) o puede tener un dispositivo virtual (**veth**) el que puede ser puesto directamente en un dispositivo de red con bridge compartido con el host. También puede ser colocado en un dispositivo de red con bridge al que se le asigna una red interna y se enmascara para acceder hacia redes externas.

En este caso se hará uso de `veth` con `host-shared bridge`. Suponiendo que la interfaz de red se llama `enp2s0`, se debe crear los archivos `/etc/netctl/enp2s0` y `/etc/netctl/vmbr0` como sigue:

```
Description='Ethernet Interface'
Interface=enp2s0
Connection=ethernet
IP=no
```

```
Description='Bridge Interface'
Interface=vmbr0
Connection=bridge
BindsToInterfaces=enp2s0
IP=static
Address=('192.168.0.1/24')
Gateway='192.168.0.254'
DNS=('8.8.8.8 8.8.4.4')
Hostname="host.dominio.tld"
DNSDomain="dominio.tld"
DNSSearch="dominio.tld"
SkipNoCarrier=yes
SkipForwardingDelay=yes
```

En la configuración anterior se hace uso de una dirección de red estática en el `bridge`, de existir un servidor DHCP se debe configurar como sigue:

```
Description='Bridge Interface'
Interface=vmbr0
Connection=bridge
BindsToInterfaces=enp2s0
IP=dhcp
SkipNoCarrier=yes
SkipForwardingDelay=yes
```

Activar las interfaces de red.

```
systemctl stop lxc-net
systemctl disable lxc-net
netctl stop-all
netctl enable enp2s0 vmbr0
```

Reiniciar el host.

La sección `network` en la configuración del contenedor, almacenada en el host en `/var/lib/lxc/<nombre-del-contenedor>/config` luciría como sigue:

```
lxc.net.0.type = veth
lxc.net.0.hwaddr = 00:16:3e:c6:64:cc
lxc.net.0.ipv4.address = 192.168.0.1/24
lxc.net.0.ipv4.gateway = 192.168.0.254
lxc.net.0.link = vmbr0
lxc.net.0.flags = up
```

Es posible también utilizar una plantilla para los nuevos contenedores. Editar el archivo `/etc/lxc/default.conf` como sigue:

```
lxc.net.0.type = veth
lxc.net.0.link = vmbr0
lxc.net.0.flags = up
lxc.net.0.hwaddr = 00:16:3e:xx:xx:xx
```

## Administración de contenedores

A diferencia del paquete LXC en Debian/Ubuntu, el disponible en Arch Linux no contiene las plantillas de creación y configuración de contenedores distintos de la propia distribución; sin embargo, es posible agregarlos. Para ello, se debe extraer el contenido del paquete `lxc-templates` de Debian/Ubuntu en los subdirectorios existentes en `/usr/share/lxc/config` y `/usr/share/lxc/templates`.

Otra opción es descargar los fuentes del sitio oficial [LXC Downloads](https://linuxcontainers.org/lxc/downloads/), compilarlos e instalarlos en el sistema.

### Crear contenedores

Los contenedores se descargan utilizando `debootstrap`, posteriormente un sistema Debian mínimo se instala en `/var/lib/lxc/<nombre-del-contenedor>/rootfs` listo para ejecutarse y ser usado.

```
lxc-create -n <nombre-del-contenedor> -t debian -- -r buster
```

Si se accede a Internet a través de un proxy, es posible especificarlo en variables de entorno para descargar los archivos del contenedor.

```
export http_proxy=http://usuario:contraseña@proxy:puerto/
export https_proxy=http://usuario:contraseña@proxy:puerto/
```

Si se tiene un mirror del repositorio de paquetes de Debian/Ubuntu también es posible pasar los parámetros `--mirror=MIRROR` y `--security-mirror=MIRROR` quedando así:

```
lxc-create -n <nombre-del-contenedor> -t debian -- -r buster --mirror=http//debian.dominio.tld/buster --security-mirror=http//debian.dominio.tld/buster-security
```

Si se desea crear el contenedor con un tamaño específico de dispositivo de bloques, es decir emulando un disco duro.

```
lxc-create -n <nombre-del-contenedor> -t debian -B loop --fstype ext4 --fssize 5G -- -r buster --mirror=http//debian.dominio.tld/buster --security-mirror=http//debian.dominio.tld/buster-security
```

### Operaciones básicas con contenedores

Iniciar un contenedor en primer plano

```
lxc-start -n <nombre-del-contenedor> -F
```

Iniciar un contenedor en segundo plano

```
lxc-start -n <nombre-del-contenedor>
```

Para iniciar automáticamente durante el arranque del host, editar la configuración del contenedor añadiendo `lxc.start.auto = 1`.

Ejecutar una consola en el contenedor

```
lxc-console -n <nombre-del-contenedor> -t 0
```

Detener un contenedor enviando un apagado limpio

```
lxc-stop -n <nombre-del-contenedor>
```

Detener un contenedor terminando todos los procesos activos

```
lxc-stop -k -n <nombre-del-contenedor>
```

Clonar un contenedor

```
lxc-copy -n <nombre-del-contenedor> -N <nombre-nuevo-contenedor> -s
```

Eliminar un contenedor

```
lxc-destroy -n <nombre-del-contenedor> -f
```

Listar contenedores

```
lxc-ls --fancy
```

Cambiar contraseña de root

```
lxc-attach -n <nombre-del-contenedor> passwd
```

### TIP

Los contenedores que se creen usando las plantillas predefinidas en el paquete `lxc-templates` son muy básicos, de hecho se reducen a la instalación de los siguientes paquetes:

Extracto de fichero plantilla `/usr/share/lxc/templates/lxc-debian`.

```
    packages=\
$init,\
ifupdown,\
locales,\
dialog,\
isc-dhcp-client,\
netbase,\
net-tools,\
$iproute,\
openssh-server
```

Esto puede ocasionar molestias a la hora de la instalación y configuración de servicios dentro de los contenedores, porque como se puede observar no se instala ningún editor de texto.

Una opción sería modificar el fichero plantilla de la distribución a utilizar agregando paquetes al listado predefindo. Sin embargo, esta solución no es idónea para usuarios noveles quienes por lo general desconocen qué paquetes son los encargados de hacer funcionar muchos de los comandos que se ejecutan en la consola; para ellos, la solución ideal sería solo agregar al listado, el paquete `aptitude` y una vez creado y teniendo funcionando el contenedor, ejecutar la selección de tareas de instalación de paquetes que dejarían el sistema con una configuración mínima completamente operativa.

```
aptitude install -y ~pstandard ~prequired ~pimportant
```

## Conclusiones

El objetivo de esta sencilla guía es proporcionar un entorno LXC funcional sobre Arch Linux para ejecutar contenedores con distribuciones de Debian/Ubuntu con fines educativos; para entornos productivos, se deberían realizar configuraciones más avanzadas. Se recomienda consultar el manual de cada una de las herramientas disponibles en LXC.

## Referencias

* [Linux Containers](https://linuxcontainers.org/)
* [Linux Containers - LXC - Introduction](https://linuxcontainers.org/lxc/introduction/)
* [Instalación y configuración de LXC en Debian Stretch](https://www.sysadminsdecuba.com/2019/08/instalacion-y-configuracion-de-lxc-en-debian-stretch/)
* [Linux Containers - ArchWiki](https://wiki.archlinux.org/index.php/Linux_Containers)
* [Network configuration - ArchWiki](https://wiki.archlinux.org/index.php/Network_configuration)
* [Network bridge - ArchWiki](https://wiki.archlinux.org/index.php/Network_bridge)
* [LXC 2.0 container not starting on Debian 9 Stretch when using cgroup limits](https://www.claudiokuenzler.com/blog/832/lxc-2.0-container-debian-9-stretch-not-starting-using-cgroup-limits)
* [man 5 lxc.container.conf]
