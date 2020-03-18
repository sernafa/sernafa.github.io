---
title:      "Mikrotik, configuración IPSec site-to-site"
date:       2020-02-29 19:00:04
excerpt:    "Proceso de configuración de un tunel IPSec site-to-site con routers Mikrotik"
tags:       mikrotik network vpn ipsec
header:
    teaser: /assets/images/mikrotik_teaser.png
---

La configuración de IPSec site-to-site descrita en el siguiente ejemplo, hace uso de la versión de mikrotik 6.46 o superior.  Las reglas del firewall son de ambito general, para cada instalación habria que realizar los ajustes necesarios.

## Conceptos IPSec en Mikrotik

El orden lógico de configuración de los elementos sería el siguiente:

*IPSec Profile -> IPSec Proposal -> IPSec Peer -> IPSec Identity -> IPSec Policy*

* [IPSec Profile](https://wiki.mikrotik.com/wiki/Manual:IP/IPsec#Profiles), encriptación de la Phase 1.
* [IPSec Proposal](https://wiki.mikrotik.com/wiki/Manual:IP/IPsec#Proposals), encriptación de la Phase 2.
* [Peer](https://wiki.mikrotik.com/wiki/Manual:IP/IPsec#Peers) & [Identity](https://wiki.mikrotik.com/wiki/Manual:IP/IPsec#Identities), definición del remoto.
* [Policy](https://wiki.mikrotik.com/wiki/Manual:IP/IPsec#Policies), definición del dominio de encriptación entre los dos remotos.

## Configuración del firewall

* Permitir protocolos VPN IPSEC (IP Protocol ID 50 (ESP), IP Protocol ID 51 (AH), UDP Port 500 (IKE), UDP Port 4500) desde la IP externa de cada parte del tunel, en el ejemplo es 1.2.3.4.

```shell
/ip firewall
add action=accept chain=input src-address=1.2.3.4 protocol=ipsec-esp add action=accept chain=input src-address=1.2.3.4 protocol=ipsec-ah add action=accept chain=input src-address=1.2.3.4 dst-port=500 protocol=udp
add action=accept chain=input src-address=1.2.3.4 dst-port=4500 protocol=udp
add action=accept chain=output dst-address=1.2.3.4
```

> Nota: una vez agregadas las reglas se debe verificar el orden, deben estar antes de la regla que deniega

## Configurar IPSec

### Phase 1

Si usamos un [RB750Gr3 (hEX)](https://mikrotik.com/product/RB750Gr3), si revisamos el soporte de algoritmos de encriptacion en la [Wiki de Mikrotik](https://wiki.mikrotik.com/wiki/Manual:IP/IPsec#Site_to_Site_IPsec_tunnel), hacemos la siguiente selección para usar la capacidad de hardware de este hardware.

```shell
/ip ipsec profile
add dh-group=modp1024 enc-algorithm=aes-256,3des,des hash-algorithm=sha256 name=phase1
```

### Phase 2

Realizamos la configuración siguiendo la misma recomendación.

```shell
/ip ipsec proposal
add auth-algorithms=sha256,sha1,md5 enc-algorithms=aes-256-cbc,aes-128-cbc,3des lifetime=1d pfs-group=none name=phase2
```

### Configuración del PEER

Configuramos el peer, con su *phase1*.

```shell
/ip ipsec peer
add address=1.2.3.4/32 name=VDC profile=phase1
/ip ipsec identity
add peer=VDC secret=PRE_SHARED_KEY
```

### Política de Encriptación

Si el rotuer remoto, tiene en su interfaz WAN directamente la IP pública, realizaremos la siguiente configuración con su *phase2*:

```shell
/ip ipsec policy
add dst-address=192.168.100.0/24 peer=VDC proposal=phase2 sa-dst-address=1.2.3.4 sa-src-address=9.8.7.6 src-address=192.168.2.0/24 tunnel=yes
```

Si por el contrario, esta detras de un router NAT, con una red intermedia, por ejemplo la IP real del router Mikrotik WAN es 192.168.11.3, lo haremos así:

```shell
/ip ipsec policy
add dst-address=192.168.100.0/24 peer=VDC proposal=phase2 sa-dst-address=1.2.3.4 sa-src-address=192.168.11.3 src-address=192.168.2.0/24 tunnel=yes
```

## Verificar la conexión

```shell
/ip ipsec active-peers
print
Debe mostrar el peer activo:
Flags: R – responder, N – natt-peer
# ID STATE UPTIME PH2-TOTAL REMOTE-ADDRESS DYNAMIC-ADDRESS
0 established 5m17s 1 1.2.3.4
```

```shell
/ip ipsec installed-sa
print
```

## Configuración del firewall

Si consultamos la [Wiki de Mikrotik](https://wiki.mikrotik.com/wiki/Manual:IP/IPsec#Site_to_Site_IPsec_tunnel), podemos ver como se realiza la configuración de las tablas firewall, para poder permitir que el trafico viaje entre ambas redes.  Hay dos formas, con la tabla NAT, sería la función normal.  O aprovechar la caracteristica [IP/Firewall/Raw](https://wiki.mikrotik.com/wiki/Manual:IP/Firewall/Raw), que permite hacer un bypass del *connection tracking*, de forma que descargamos la CPU del propio router:

```shell
/ip firewall raw
add action=notrack chain=prerouting src-address=192.168.100.0/24 dst-address=192.168.2.0/24
add action=notrack chain=prerouting src-address=192.168.2.0/24 dst-address=192.168.100.0/24
```

### Enlaces

* [Configurar VPN IPSEC a un Mikrotik en un Datacenter Virtual](https://blog.baehost.com/configurar-vpn-ipsec-en-un-datacenter-virtual-a-un-mikrotik/)
