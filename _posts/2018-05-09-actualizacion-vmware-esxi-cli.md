---
layout: post
title: "Actualización de VMware ESXi desde la CLI"
date: 2018-05-09 11:21:29 +0200
categories: [VMware, ESXi]
tags: [vmware, esxi, virtualization, cli]
description: "Instalación de parches y perfiles de VMware ESXi desde la línea de comandos, utilizando el repositorio en línea o un bundle offline."
---

Actualizar VMware ESXi desde la línea de comandos es un proceso sencillo y rápido. Nos permite instalar parches de forma remota, sin necesidad de acceder físicamente al host ni utilizar una interfaz gráfica.

Podemos realizar la actualización directamente desde el repositorio de VMware o utilizar un *offline bundle* almacenado en un datastore. En ambos casos, antes de aplicar el parche debemos comprobar las máquinas virtuales que siguen en ejecución y poner el host en modo mantenimiento.

## Consultar las versiones disponibles

Podemos utilizar estos recursos para localizar versiones y perfiles de actualización:

- [VMware ESXi Patch Tracker](https://esxi-patches.v-front.de/)
- [Búsqueda de parches de VMware](https://my.vmware.com/group/vmware/patch#search)

El nombre exacto del perfil es importante, ya que tendremos que indicarlo posteriormente en el comando de actualización.

## Comprobar las máquinas virtuales en ejecución

Antes de poner el host en mantenimiento, comprobamos si todavía existen máquinas virtuales en ejecución:

```shell
esxcli vm process list
```

Este comando muestra las máquinas virtuales activas y la información necesaria para identificarlas. Antes de continuar debemos apagarlas o migrarlas a otro host, según el entorno.

## Activar el modo mantenimiento

Cuando el host ya no ejecuta ninguna carga, activamos el modo mantenimiento:

```shell
esxcli system maintenanceMode set --enable true
```

De esta forma evitamos que se inicien máquinas virtuales mientras estamos modificando el software del hipervisor.

## Actualización desde el repositorio en línea

Si el host puede acceder a Internet, habilitamos temporalmente las conexiones HTTP salientes en el firewall de ESXi:

```shell
esxcli network firewall ruleset set -e true -r httpClient
```

### Listar los perfiles disponibles

Consultamos los perfiles publicados en el repositorio de VMware:

```shell
esxcli software sources profile list -d https://hostupdate.vmware.com/software/VUM/PRODUCTION/main/vmw-depot-index.xml
```

La opción `-d` indica el *depot* que contiene las imágenes disponibles. La salida puede ser extensa, por lo que debemos localizar el nombre exacto del perfil que queremos instalar.

### Aplicar el perfil seleccionado

Una vez identificado el perfil, ejecutamos la actualización indicando el repositorio y su nombre:

```shell
esxcli software profile update -d https://hostupdate.vmware.com/software/VUM/PRODUCTION/main/vmw-depot-index.xml -p ESXi-6.0.0-20180304001-standard
```

La opción `-p` selecciona el perfil concreto dentro del repositorio. Antes de ejecutar el comando debemos comprobar que corresponde con la versión y el hardware del host que estamos actualizando.

## Actualización mediante un offline bundle

Cuando el host no dispone de acceso a Internet, o cuando necesitamos utilizar una imagen personalizada por el fabricante, podemos descargar el bundle previamente y dejarlo en un datastore accesible desde ESXi.

En este ejemplo utilizaremos una imagen de Fujitsu almacenada en un volumen NFS:

```text
/vmfs/volumes/NFS/VMWARE/VMware-ESXi-6.5.0.update01-7967591-Fujitsu-v412-1-offline_bundle.zip
```

### Consultar los perfiles incluidos

Antes de instalar el bundle, listamos los perfiles que contiene:

```shell
esxcli software sources profile list -d /vmfs/volumes/NFS/VMWARE/VMware-ESXi-6.5.0.update01-7967591-Fujitsu-v412-1-offline_bundle.zip
```

Esto nos permite confirmar el nombre del perfil proporcionado por el fabricante.

### Instalar el perfil

Aplicamos la actualización utilizando la ruta del bundle y el perfil seleccionado:

```shell
esxcli software profile update -d /vmfs/volumes/NFS/VMWARE/VMware-ESXi-6.5.0.update01-7967591-Fujitsu-v412-1-offline_bundle.zip -p Fujitsu-VMvisor-Installer-6.5-7967591-v412-1
```

El procedimiento es el mismo que con el repositorio en línea. La diferencia es que el *depot* se encuentra en un fichero local y puede contener una imagen adaptada específicamente al hardware del servidor.

## Resumen

El flujo de actualización queda dividido en cuatro pasos:

1. Comprobar y detener o migrar las máquinas virtuales.
2. Poner el host ESXi en modo mantenimiento.
3. Consultar los perfiles disponibles en el repositorio o en el bundle offline.
4. Aplicar el perfil correspondiente mediante `esxcli software profile update`.

Utilizar la CLI facilita repetir el procedimiento y trabajar de la misma forma tanto con las imágenes estándar de VMware como con las imágenes personalizadas por el fabricante.
