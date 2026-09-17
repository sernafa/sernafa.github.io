---
layout: post
title: "Photon OS 2: configuración básica"
date: 2018-03-22
categories: [VMware, Linux]
tags: [vmware, photon-os, linux, networking]
description: "Configuración básica de Photon OS 2: red estática, gestión de paquetes y distribución del teclado."
---

Photon OS es una distribución Linux ligera desarrollada por VMware y orientada a la ejecución de contenedores. Su instalación es mínima, por lo que algunas tareas habituales se realizan de forma diferente a otras distribuciones Linux.

Este artículo recoge la configuración básica que utilicé con **Photon OS 2**: asignar una dirección IP estática, actualizar el sistema, instalar paquetes y configurar el teclado.

## Configurar una dirección IP estática

Photon OS utiliza `systemd-networkd` para gestionar la configuración de red. Para asignar una dirección estática creamos o modificamos el siguiente fichero:

```text
/etc/systemd/network/99-static-en.network
```

Su contenido será similar a este:

```ini
[Match]
Name=e*

[Network]
DHCP=no
Address=192.168.34.191/24
Gateway=192.168.34.1
DNS=192.168.34.1 192.168.34.30
Domains=domain.com
NTP=0.pool.ntp.org 1.pool.ntp.org
```

La sección `[Match]` determina las interfaces a las que se aplicará la configuración. En este caso, `Name=e*` selecciona las interfaces cuyo nombre comienza por `e`.

La sección `[Network]` contiene los parámetros de red:

- `DHCP=no` desactiva la configuración mediante DHCP.
- `Address` establece la dirección IP y su máscara.
- `Gateway` define la puerta de enlace predeterminada.
- `DNS` contiene los servidores utilizados para la resolución de nombres.
- `Domains` establece el dominio de búsqueda.
- `NTP` indica los servidores utilizados para sincronizar la hora.

Antes de guardar el fichero debemos adaptar las direcciones y el nombre de la interfaz a nuestro entorno.

## Gestionar paquetes con tdnf

Photon OS utiliza `tdnf` como gestor de paquetes. Su funcionamiento es similar al de `dnf` o `yum`, pero está pensado para mantener una instalación ligera.

Para actualizar los paquetes instalados:

```shell
tdnf upgrade
```

Para instalar un paquete, por ejemplo `wget`:

```shell
tdnf install wget
```

De esta forma podemos partir de la instalación mínima de Photon OS e incorporar solamente las herramientas que necesitemos.

## Configurar el teclado

La instalación mínima no incluye necesariamente los recursos necesarios para cambiar la distribución del teclado. Primero instalamos el paquete `kbd`:

```shell
tdnf install kbd
```

Después configuramos la distribución española:

```shell
localectl set-keymap es
```

Podemos consultar la configuración aplicada ejecutando:

```shell
localectl status
```

## Resumen

Con estos pasos dejamos preparada una instalación básica de Photon OS 2:

1. Definimos una dirección IP estática mediante `systemd-networkd`.
2. Actualizamos el sistema e instalamos herramientas con `tdnf`.
3. Añadimos el paquete de teclado y seleccionamos la distribución española.

Photon OS parte de una base reducida, pero mantiene las herramientas necesarias para configurar el sistema de forma sencilla desde la terminal.
