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
pacman -S lxc
```

El paquete `lxc` sugiere la instalación de `dnsmasq`, necesario para el correcto funcionamiento de la herramienta `lxc-net`, que permite configurar un bridge simple para los contenedores. Sin embargo, podemos prescindir de esta herramienta como se verá más adelante. Esta guía utiliza el modelo de red en el que se usa un bridge compartido entre el host y los guests.

Para poder crear contenedores de Debian/Ubuntu, debemos instalar además:

```
pacman -S debootstrap debian-archive-keyring ubuntu-keyring
```

## Configuración de la red

El paquete de Arch Linux no incluye ninguna configuración de red por defecto para los contenedores. El archivo `/etc/lxc/default.conf` contiene solamente el valor `lxc.network.type = empty`.

A un contenedor típico se le puede asignar su propia interface de red física (**phys**) o puede tener un dispositivo virtual (**veth**) el que puede ser puesto directamente en un dispositivo de red con bridge compartido con el host. También puede ser colocado en un dispositivo de red con bridge al que se le asigna una red interna y se enmascara para acceder hacia redes externas.

En este caso se hará uso de `veth` con `host-shared bridge`. Suponiendo que la interfaz de red se llama `enp2s0`, se debe crear los archivos `/etc/netctl/enp2s0` y `/etc/netctl/bridge` como sigue: 

```
Description='Ethernet Interface'
Interface=enp2s0
Connection=ethernet
IP=no
```

```
Description='Bridge Interface'
Interface=br0
Connection=bridge
BindsToInterfaces=enp2s0
IP=static
Address=('192.168.0.100/24')
Gateway='192.168.0.1'
DNS=('192.168.0.1')
Hostname="host.dominio.tld"
DNSDomain="dominio.tld"
DNSSearch="dominio.tld"
SkipNoCarrier=yes
SkipForwardingDelay=yes
```

En la configuración anterior se hace uso de una dirección de red estática en el `bridge`, de existir un servidor DHCP se debe configurar como sigue:

```
Description='Bridge Interface'
Interface=br0
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
netctl enable enp2s0 bridge
```

Reiniciar el host.

La sección `network` en la configuración del contenedor, almacenada en el host en `/var/lib/lxc/nombre-del-contenedor/config` luciría como sigue:

```
lxc.net.0.type = veth
lxc.net.0.hwaddr = 00:16:3e:c6:64:cc
lxc.net.0.ipv4.address = 192.168.0.200/24
lxc.net.0.ipv4.gateway = 192.168.0.1
lxc.net.0.link = br0
lxc.net.0.flags = up
```

Es posible también utilizar una plantilla para los nuevos contenedores. Editar el archivo `/etc/lxc/default.conf` como sigue:

```
lxc.net.0.type = veth
lxc.net.0.link = br0
lxc.net.0.flags = up
lxc.net.0.hwaddr = 00:16:3e:xx:xx:xx
```

## Administración de contenedores

A diferencia del paquete LXC en Debian/Ubuntu, el disponible en Arch Linux no contiene las plantillas de creación y configuración de contenedores distintos de la propia distribución; sin embargo, es posible agregarlos. Para ello, se debe extraer el contenido del paquete `lxc-templates` de Debian/Ubuntu en los subdirectorios existentes en `/usr/share/lxc/config` y `/usr/share/lxc/templates`.

### Crear contenedores

Los contenedores se descargan utilizando `debootstrap`, posteriormente un sistema Debian mínimo se instala en `/var/lib/lxc/<nombre-del-contenedor>/rootfs` listo para ejecutarse y ser usado.

```
lxc-create -n <nombre> -t debian -- -r buster
```

Si se accede a Internet a través de un proxy, es posible especificarlo en variables de entorno para descargar los archivos del contenedor.

```
export http_proxy=http://usuario:contraseña@proxy:puerto/
export https_proxy=http://usuario:contraseña@proxy:puerto/
```

Si se tiene un mirror del repositorio de paquetes de Debian/Ubuntu también es posible pasar los parámetros `--mirror=MIRROR` y `--security-mirror=MIRROR` quedando así:

```
lxc-create -n <nombre> -t debian -- -r buster --mirror=http//debian.dominio.tld/buster --security-mirror=http//debian.dominio.tld/buster-security
```

### Iniciar y detener contenedores

Iniciar un contenedor en segundo plano

```
lxc-start -n nombre-del-contenedor
```

Ejecutar una consola en el contenedor

```
lxc-console -n nombre-del-contenedor
```

Detener un contenedor enviando un apagado limpio

```
lxc-stop -n nombre-del-contenedor
```

Detener un contenedor terminando todos los procesos activos

```
lxc-stop -k -n nombre-del-contenedor
```

Para iniciar automáticamente durante el arranque del host, editar la configuración del contenedor añadiendo `lxc.start.auto = 1`.

## Listar contenedores

```
lxc-ls --fancy
```

## Cambiar contraseña de root

```
lxc-attach -n <nombre-del-contenedor> passwd
```

## Conclusiones

El objetivo de esta sencilla guía es proporcionar un entorno LXC funcional sobre Arch Linux para ejecutar contenedores con distribuciones de Debian/Ubuntu. Se recomienda consultar el manual de cada una de las herramientas disponibles en LXC.

## Referencias

* [Linux Containers](https://linuxcontainers.org/)
* [Linux Containers - LXC - Introduction](https://linuxcontainers.org/lxc/introduction/)
* [Linux Containers - ArchWiki](https://wiki.archlinux.org/index.php/Linux_Containers)
* [Network configuration - ArchWiki](https://wiki.archlinux.org/index.php/Network_configuration)
* [Network bridge - ArchWiki](https://wiki.archlinux.org/index.php/Network_bridge)
