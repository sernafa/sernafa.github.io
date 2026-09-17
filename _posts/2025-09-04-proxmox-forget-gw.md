---
layout: post
title: "El nodo Proxmox pierde la puerta de enlace predeterminada al reiniciarse"
date: 2025-09-04 15:29
categories: Proxmox Redes
tags: network linux
---

# El problema

Cuando un nodo Proxmox arranca y **la ruta predeterminada no aparece**, aunque la dirección IP de gestión sí esté configurada, normalmente encontrarás este mensaje en los registros:

```text
Error: Nexthop device is not up.
```

### Qué está ocurriendo realmente

En sistemas administrados mediante **ifupdown2** que utilizan interfaces apiladas (`bond` → puente → VLAN), puede producirse una condición de carrera: el sistema intenta instalar la ruta predeterminada **antes** de que la interfaz del siguiente salto esté completamente activa.

De forma predeterminada, ifupdown2 **retrasa los cambios de estado de las interfaces subordinadas hasta que cambia el estado de la interfaz principal**. Como consecuencia, el dispositivo del siguiente salto puede seguir inactivo en el momento de aplicar la ruta.

### La solución directa

Indica a ifupdown2 que **no vincule los estados administrativos de las interfaces principal y subordinadas**. Para ello, establece `link_master_slave=0`.

1. Edita el archivo de configuración:

```text
/etc/network/ifupdown2/ifupdown2.conf
```

2. Añade esta línea o modifica su valor si ya existe:

```text
link_master_slave=0
```

3. Reinicia el servicio de red:

```bash
systemctl restart networking
```

### Por qué funciona

Al desactivar la vinculación entre la interfaz principal y las subordinadas, los dispositivos de nivel inferior pueden activarse de forma independiente y estar disponibles a tiempo para instalar la ruta. Así se elimina el intervalo durante el cual el kernel rechaza la ruta predeterminada con el mensaje `Nexthop device is not up`.
