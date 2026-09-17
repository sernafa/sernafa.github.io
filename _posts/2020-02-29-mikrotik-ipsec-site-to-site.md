---
layout: post
title: "MikroTik: configuración de una VPN IPsec site-to-site"
date: 2020-02-29 19:00:04 +0100
categories: [MikroTik, Networking]
tags: [mikrotik, network, vpn, ipsec]
description: "Configuración de un túnel VPN IPsec site-to-site entre dos redes mediante routers MikroTik."
---

Una VPN IPsec *site-to-site* permite comunicar dos redes privadas a través de Internet como si existiera un enlace directo entre ellas.

En este ejemplo vamos a configurar uno de los extremos del túnel utilizando un router MikroTik. La configuración se realizó con RouterOS 6.46 y utiliza reglas de firewall de carácter general. En una instalación real tendremos que adaptarlas a la topología, las direcciones IP y la política de seguridad de cada entorno.

## Topología del ejemplo

Para poder seguir la configuración utilizaremos estos valores:

| Elemento | Dirección |
|---|---|
| Red local | `192.168.2.0/24` |
| IP pública local | `9.8.7.6` |
| Red remota | `192.168.100.0/24` |
| IP pública remota | `1.2.3.4` |

Cuando el MikroTik se encuentre detrás de otro router con NAT, utilizaremos `192.168.11.3` como dirección de su interfaz WAN.

## Elementos de una configuración IPsec

Antes de comenzar conviene entender el orden lógico de los elementos que vamos a configurar:

```text
IPsec Profile → IPsec Proposal → IPsec Peer → IPsec Identity → IPsec Policy
```

- **IPsec Profile:** define los algoritmos utilizados durante la fase 1 de la negociación.
- **IPsec Proposal:** define los algoritmos de la fase 2.
- **Peer e Identity:** identifican el extremo remoto y establecen la clave compartida.
- **Policy:** determina qué redes forman parte del dominio de cifrado.

El perfil y la propuesta establecen cómo se protegerá la conexión. El *peer* indica con quién queremos establecerla y la política especifica qué tráfico debe atravesar el túnel.

## Permitir IPsec en el firewall

El primer paso es permitir el tráfico necesario para establecer la VPN desde la IP pública del extremo remoto, `1.2.3.4` en este ejemplo.

Necesitaremos permitir ESP, AH, IKE mediante UDP 500 y, cuando existe NAT entre los extremos, NAT-T mediante UDP 4500.

```shell
/ip firewall filter
add action=accept chain=input src-address=1.2.3.4 protocol=ipsec-esp
add action=accept chain=input src-address=1.2.3.4 protocol=ipsec-ah
add action=accept chain=input src-address=1.2.3.4 dst-port=500 protocol=udp
add action=accept chain=input src-address=1.2.3.4 dst-port=4500 protocol=udp
add action=accept chain=output dst-address=1.2.3.4
```

> El orden de las reglas es importante. Las reglas que permiten IPsec deben situarse antes de cualquier regla general que deniegue ese tráfico.

## Configurar IPsec

La configuración se divide en las dos fases de la negociación y, posteriormente, en la definición del extremo remoto y de las redes que queremos comunicar.

### Fase 1: IPsec Profile

Para este laboratorio se utilizó un [MikroTik RB750Gr3 (hEX)](https://mikrotik.com/product/RB750Gr3). La selección de algoritmos se hizo teniendo en cuenta las capacidades de aceleración por hardware disponibles en este equipo.

Creamos el perfil de la fase 1:

```shell
/ip ipsec profile
add dh-group=modp1024 enc-algorithm=aes-256,3des,des hash-algorithm=sha256 name=phase1
```

Este perfil se llamará `phase1` y lo asignaremos al *peer* remoto más adelante.

### Fase 2: IPsec Proposal

La propuesta define los parámetros utilizados para proteger el tráfico entre las dos redes una vez establecido el canal de negociación.

```shell
/ip ipsec proposal
add auth-algorithms=sha256,sha1,md5 enc-algorithms=aes-256-cbc,aes-128-cbc,3des lifetime=1d pfs-group=none name=phase2
```

La propuesta queda guardada con el nombre `phase2` y será utilizada por la política IPsec.

### Peer e Identity

Ahora indicamos la dirección del extremo remoto y asociamos el perfil creado para la fase 1:

```shell
/ip ipsec peer
add address=1.2.3.4/32 name=VDC profile=phase1
```

La identidad vincula el *peer* con la clave precompartida utilizada por ambos extremos:

```shell
/ip ipsec identity
add peer=VDC secret=PRE_SHARED_KEY
```

`PRE_SHARED_KEY` debe sustituirse por la misma clave segura configurada en el router remoto.

### Política de cifrado

La política determina qué tráfico debe entrar en el túnel. En este caso queremos comunicar la red local `192.168.2.0/24` con la red remota `192.168.100.0/24`.

Si el router dispone directamente de la IP pública en su interfaz WAN, la política será:

```shell
/ip ipsec policy
add dst-address=192.168.100.0/24 peer=VDC proposal=phase2 sa-dst-address=1.2.3.4 sa-src-address=9.8.7.6 src-address=192.168.2.0/24 tunnel=yes
```

En cambio, si el MikroTik se encuentra detrás de otro router que realiza NAT y su interfaz WAN tiene la dirección `192.168.11.3`, utilizaremos esa dirección como `sa-src-address`:

```shell
/ip ipsec policy
add dst-address=192.168.100.0/24 peer=VDC proposal=phase2 sa-dst-address=1.2.3.4 sa-src-address=192.168.11.3 src-address=192.168.2.0/24 tunnel=yes
```

La diferencia entre ambos casos está en la dirección desde la que el MikroTik origina la asociación de seguridad. El dominio de cifrado continúa siendo el mismo: `192.168.2.0/24` en el extremo local y `192.168.100.0/24` en el remoto.

## Verificar la conexión

Podemos comprobar el estado del *peer* con:

```shell
/ip ipsec active-peers print
```

Cuando la fase 1 se ha establecido correctamente veremos una salida similar a esta:

```text
Flags: R - responder, N - natt-peer
 # ID  STATE        UPTIME  PH2-TOTAL  REMOTE-ADDRESS
 0     established  5m17s   1          1.2.3.4
```

Después podemos consultar las asociaciones de seguridad instaladas:

```shell
/ip ipsec installed-sa print
```

Que el *peer* aparezca como `established` confirma que la negociación se ha completado, pero no garantiza por sí solo que el tráfico esté circulando. También debemos comprobar las rutas, las reglas de firewall y los contadores de las asociaciones de seguridad mientras generamos tráfico entre ambas redes.

## Permitir el tráfico entre las redes

Además de permitir la negociación IPsec, el firewall debe dejar pasar el tráfico entre las redes local y remota.

Una posibilidad es tratar las excepciones necesarias mediante la tabla NAT. Otra es utilizar `raw` para excluir este tráfico del *connection tracking*. En el ejemplo se utiliza la segunda opción:

```shell
/ip firewall raw
add action=notrack chain=prerouting src-address=192.168.100.0/24 dst-address=192.168.2.0/24
add action=notrack chain=prerouting src-address=192.168.2.0/24 dst-address=192.168.100.0/24
```

Las dos reglas cubren ambos sentidos de la comunicación. Al aplicar `notrack`, el router evita que estas conexiones pasen por el seguimiento de estado, reduciendo el trabajo que debe realizar para este tráfico.

Como ocurre con las reglas anteriores, debemos revisar su posición dentro del firewall y adaptarlas a las redes reales de cada extremo.

## Resumen

La configuración completa sigue una secuencia sencilla:

1. Permitimos la negociación IPsec en el firewall.
2. Definimos los algoritmos de las fases 1 y 2.
3. Configuramos el *peer* remoto y su clave compartida.
4. Creamos la política que relaciona las dos redes privadas.
5. Permitimos el tráfico entre ambas redes y verificamos las asociaciones de seguridad.

Separar la configuración en estos bloques facilita localizar los problemas: primero comprobamos que el *peer* negocia, después que se instala la fase 2 y, por último, que el tráfico coincide con la política y atraviesa el firewall.

## Recursos

- [Configurar VPN IPsec a un MikroTik en un Datacenter Virtual](https://blog.baehost.com/configurar-vpn-ipsec-en-un-datacenter-virtual-a-un-mikrotik/)
