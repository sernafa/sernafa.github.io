---
layout: post
title: "Diseñando un clúster OpenStack hiperconvergente y de alta disponibilidad"
date: 2026-11-16 00:00:00 +0100
categories: [OpenStack, Infraestructura]
tags: [openstack, kolla-ansible, ceph, ovn, alta-disponibilidad]
toc: true
description: "Diseño de un laboratorio OpenStack de tres nodos con Ceph, OVN y servicios de control en alta disponibilidad."
---

*Construyendo una nube privada con OpenStack, Kolla-Ansible y Ceph — Parte 2*

## Introducción

En el primer artículo vimos qué hacen Nova, Neutron y los demás servicios de OpenStack. Ahora vamos a ver dónde se ejecutan esos servicios y qué recursos compartirán en el laboratorio.

Si conocemos Proxmox, la idea de ejecutar máquinas virtuales y almacenar sus discos en Ceph sobre los mismos servidores nos resultará familiar. En OpenStack también tendremos que decidir dónde colocamos las API, la base de datos y la mensajería, y cómo mantenemos disponible la conectividad de las instancias.

Utilizaremos tres nodos físicos para control, cómputo, almacenamiento y red. A esta forma de agrupar funciones la llamamos **hiperconvergencia**. Veremos cómo repartimos los servicios entre los nodos, qué queda en el bastión y qué debemos tener en cuenta si perdemos un servidor.

## 1. Requisitos de arquitectura

Distribuiremos los servicios críticos entre los tres nodos para que el plano de control pueda seguir funcionando si falla uno. Ceph replicará los datos entre servidores, y Neutron con OVN nos permitirá crear redes privadas, routers virtuales y conexiones hacia el exterior.

Utilizaremos Kolla-Ansible para que el despliegue y los cambios de configuración sean reproducibles. Versionaremos los inventarios y la configuración en Git. Empezaremos por los servicios básicos y más adelante incorporaremos Octavia y Barbican.

## 2. Infraestructura física

Partimos de tres servidores con características equivalentes y una máquina virtual que utilizaremos como bastión de administración.

### Características de los nodos

| Recurso             | Configuración                          |
| ------------------- | -------------------------------------- |
| Procesadores        | 1 procesador físico por nodo           |
| Memoria RAM         | Aproximadamente 64 GB por nodo         |
| Sistema operativo   | Ubuntu Server 24.04 LTS                |
| Discos del sistema  | Discos dedicados al sistema operativo  |
| Almacenamiento Ceph | 1 SSD de aproximadamente 2 TB por nodo |
| Conectividad        | Enlaces redundantes de 10 GbE          |
| Networking          | VLAN mediante interfaces trunk         |

Dedicaremos un SSD de unos 2 TB por nodo a Ceph. Cada disco será un OSD independiente y la replicación se hará entre nodos.

La capacidad bruta total será de aproximadamente 6 TB. La capacidad útil dependerá de la política de replicación y de otros factores de almacenamiento. Por ejemplo, con tres réplicas, la capacidad teórica antes de reservas y sobrecargas sería de unos 2 TB.

Partimos de Ubuntu Server 24.04 LTS instalado y actualizado, con los enlaces redundantes ya operativos.

### ¿Por qué tres nodos?

Con tres miembros, los servicios que requieren mayoría pueden mantener el quórum con dos. Eso nos permite probar qué ocurre al perder un servidor. Para seguir operando necesitaremos además que los dos nodos restantes tengan capacidad suficiente para los servicios y las cargas que deban asumir.

## 3. Arquitectura hiperconvergente

OpenStack permite separar físicamente las funciones de control, computación, almacenamiento y networking.

Esta separación facilita el dimensionamiento y el aislamiento de cargas, pero requiere más infraestructura.

En el laboratorio agruparemos las cuatro funciones en los mismos servidores.

En cada nodo tendremos estas funciones.

- **Control.** API y servicios de coordinación de OpenStack, junto con dependencias como MariaDB y RabbitMQ.
- **Computación.** Nova Compute, libvirt y KVM para ejecutar instancias.
- **Almacenamiento.** componentes de Ceph y un OSD por nodo.
- **Networking.** Open vSwitch y componentes de OVN, incluyendo funciones de gateway hacia redes externas.

La ubicación y el número de réplicas dependerán de cada servicio.

### Distribución de roles

```text
                    NUBE PRIVADA OPENSTACK

       ┌─────────────────────────────────────┐
       │       BASTIÓN DE ADMINISTRACIÓN     │
       │                                     │
       │  Kolla-Ansible · Kolla · Registry   │
       │  Git · Herramientas de operación    │
       └──────────────────┬──────────────────┘
                          │
                   Administración
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼
┌────────────────┐ ┌────────────────┐ ┌────────────────┐
│     NODO 1     │ │     NODO 2     │ │     NODO 3     │
├────────────────┤ ├────────────────┤ ├────────────────┤
│ Control        │ │ Control        │ │ Control        │
│ Nova Compute   │ │ Nova Compute   │ │ Nova Compute   │
│ OVN Gateway    │ │ OVN Gateway    │ │ OVN Gateway    │
│ Ceph OSD × 1   │ │ Ceph OSD × 1   │ │ Ceph OSD × 1   │
└────────────────┘ └────────────────┘ └────────────────┘
        │                 │                 │
        └─────────────────┼─────────────────┘
                          │
               Redes y servicios
                  distribuidos
```

Así aprovechamos los tres servidores para todos los roles. A cambio, los servicios de control, Ceph y las instancias comparten CPU, memoria y enlaces. Una recuperación de Ceph, por ejemplo, utilizará los mismos enlaces físicos que el resto de la plataforma. Con unos 64 GB de RAM por nodo tendremos que reservar memoria para los servicios antes de decidir cuánta podemos ofrecer a las instancias.

## 4. Alta disponibilidad

Una API puede seguir escuchando y aun así ser incapaz de completar una petición si pierde acceso a la base de datos o a la mensajería. Por eso distribuiremos también esas dependencias.

| Componente       | Estrategia de disponibilidad                        |
| ---------------- | --------------------------------------------------- |
| API de OpenStack | Réplicas distribuidas y balanceo mediante HAProxy   |
| Endpoints        | Direcciones IP virtuales gestionadas con Keepalived |
| Base de datos    | MariaDB Galera distribuido                          |
| Mensajería       | RabbitMQ distribuido                                |
| Almacenamiento   | Ceph con replicación entre nodos                    |
| Networking       | OVN con componentes y gateways distribuidos         |

Configuraremos buena parte de estos mecanismos con Kolla-Ansible. Al final de la serie apagaremos un nodo y comprobaremos por separado si responden las API, si los volúmenes siguen accesibles y qué ocurre con las instancias que se ejecutaban en ese servidor.

## 5. El bastión y Kolla-Ansible

Desde el bastión desplegaremos y mantendremos OpenStack. En esa máquina virtual reuniremos las herramientas que utilizaremos durante la serie.

- Ejecutar Kolla-Ansible y otras herramientas Ansible.
- Construir imágenes de contenedores mediante Kolla.
- Alojar un registro privado Docker.
- Mantener inventarios y configuraciones versionados en Git.

Con Kolla construiremos las imágenes y podremos identificar qué versiones y dependencias contiene cada una. Kolla-Ansible utilizará esas imágenes para desplegar y configurar los servicios.

Si perdemos el bastión, los contenedores que ya están en ejecución pueden seguir funcionando mientras sus dependencias estén disponibles. Lo que perderemos es el acceso al registro y al entorno desde el que ejecutamos las automatizaciones. Eso puede impedir un despliegue o una recuperación que necesite descargar imágenes. Para el laboratorio aceptaremos esa dependencia.

## 6. Consideraciones de networking

Sobre los enlaces redundantes de 10 GbE utilizaremos VLAN para separar el tráfico. Antes de asignar direcciones, conviene identificar qué circula por cada red.

| Red                | Función                                       |
| ------------------ | --------------------------------------------- |
| Gestión            | Administración de nodos y automatización      |
| Servicios internos | Comunicación entre componentes de OpenStack   |
| Almacenamiento     | Tráfico de Ceph                               |
| Overlay OVN        | Redes virtuales mediante encapsulación Geneve |
| Provider           | Conectividad externa de las instancias        |
| API pública        | Acceso HTTPS a las API y Horizon              |

La tabla distingue funciones. La asignación de VLAN puede agrupar algunas de ellas, como la gestión y la comunicación entre servicios internos.

Las instancias utilizarán redes privadas, routers virtuales y floating IPs para obtener conectividad externa.

Publicaremos las API y Horizon mediante HTTPS desde la red de acceso público y mantendremos separado el acceso de administración.

## 7. Arquitectura de referencia

Esta es la configuración que utilizaremos como referencia para las siguientes entregas.

| Elemento                 | Tecnología o decisión                  |
| ------------------------ | -------------------------------------- |
| Arquitectura             | Hiperconvergente de tres nodos         |
| Procesamiento            | 1 procesador físico por nodo           |
| Memoria                  | Aproximadamente 64 GB por nodo         |
| Sistema operativo        | Ubuntu Server 24.04 LTS                |
| Hipervisor               | KVM mediante libvirt                   |
| Despliegue               | Kolla-Ansible                          |
| Imágenes                 | Construcción propia con Kolla          |
| Registro                 | Docker Registry en el bastión          |
| Almacenamiento           | Ceph RBD                               |
| Distribución de Ceph     | 1 OSD por nodo, 3 en total             |
| Networking               | Neutron, OVN y Open vSwitch            |
| Gateways                 | Los tres nodos                         |
| Acceso a las instancias  | Redes privadas y floating IPs          |
| Publicación de servicios | Endpoints HTTPS                        |
| Alta disponibilidad      | Distribución de los servicios críticos |

## 8. Conclusiones

Con tres nodos podemos distribuir los servicios de OpenStack y probar su comportamiento ante un fallo. Al compartir servidores, perder un nodo afecta a la vez al control, al cómputo, al almacenamiento y a la red. Por eso comprobaremos la disponibilidad de cada función y la capacidad que queda en los otros dos nodos.

En el siguiente artículo concretaremos las VLAN y el direccionamiento IP. Veremos por dónde circulará el tráfico de gestión, Ceph y OVN, cómo conectaremos las instancias a la red externa y desde dónde se podrá acceder a las API de OpenStack.
