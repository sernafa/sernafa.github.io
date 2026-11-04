---
layout: post
title: "OpenStack desde cero: arquitectura, componentes y conceptos fundamentales"
date: 2026-11-03 00:00:00 +0100
categories: [OpenStack, Infraestructura]
tags: [openstack, kolla-ansible, ceph, ovn, proxmox]
toc: true
description: "Primera entrega de una serie para construir una nube privada con OpenStack, Kolla-Ansible y Ceph."
---

*Construyendo una nube privada con OpenStack, Kolla-Ansible y Ceph — Parte 1*

## Introducción

**OpenStack** es una plataforma que permite ofrecer recursos de computación, almacenamiento y red como servicios de infraestructura. Está formada por varios proyectos que trabajan conjuntamente para crear y administrar una nube privada o pública.

Si conocemos **Proxmox VE**, muchos de los conceptos nos resultarán familiares: máquinas virtuales, hipervisores, redes, discos, imágenes y almacenamiento compartido. Esa experiencia nos proporciona una buena referencia para empezar a entender OpenStack, aunque la forma de organizar y administrar esos recursos sea diferente.

En Proxmox encontramos buena parte de estas funciones integradas en una misma plataforma. En OpenStack encontramos servicios con nombres propios: Nova, Neutron, Glance, Cinder, Keystone o Placement. Cada uno tiene una responsabilidad y se relaciona con los demás para completar operaciones que ya conocemos, como crear una máquina virtual y conectarla a una red.

En este primer artículo vamos a situar esas piezas: qué hace cada servicio, con qué función de Proxmox podemos relacionarlo y cómo interviene cuando creamos una instancia. La comparación nos servirá para explicar los conceptos, teniendo en cuenta que ambas plataformas tienen ámbitos, arquitecturas y modelos de operación diferentes.

## 1. De la virtualización a la infraestructura como servicio

La virtualización permite ejecutar varios sistemas operativos sobre un servidor físico. Plataformas como Proxmox VE añaden herramientas para administrar esos recursos, crear máquinas virtuales y gestionar clústeres.

Una plataforma **IaaS (Infrastructure as a Service)** añade un modelo de consumo. El usuario solicita, por ejemplo, una instancia de cuatro vCPU y 8 GB de RAM, un volumen persistente de 100 GB y una red privada. No necesita saber qué host físico ejecutará la instancia ni qué dispositivo de almacenamiento utilizará.

Desde el punto de vista de quien administra la plataforma, esto tiene varias implicaciones:

- **Autoservicio:** los usuarios aprovisionan recursos dentro de sus permisos.
- **Multitenencia:** distintos proyectos comparten infraestructura manteniendo aislamiento lógico.
- **Abstracción:** se consume capacidad, no servidores físicos concretos.
- **Automatización:** las operaciones se exponen mediante API y pueden integrarse con Terraform, Ansible o pipelines.
- **Cuotas:** se controla cuánto puede consumir cada proyecto.

Por debajo seguiremos teniendo servidores y un hipervisor. En nuestro laboratorio utilizaremos KVM. Lo que añadimos es una capa que permite solicitar esos recursos, aplicar permisos y cuotas y coordinar su creación sin que el usuario tenga que intervenir en cada host.

## 2. OpenStack y Proxmox: dos aproximaciones diferentes

Proxmox VE integra administración de clúster, máquinas virtuales, almacenamiento, redes, autenticación e interfaz web en una plataforma. OpenStack reparte esas responsabilidades entre servicios independientes.

Cuando un usuario crea una instancia, **Keystone** interviene en la identidad, **Nova** coordina la computación, **Placement** informa de recursos disponibles, **Neutron** proporciona conectividad y **Glance** ofrece la imagen. **Cinder** participa si se emplean volúmenes gestionados.

La diferencia está en cómo se organizan y se exponen esas funciones. OpenStack las ofrece mediante servicios con sus propias API, dentro de un modelo de proyectos, permisos y cuotas. El usuario trabaja con instancias, redes y volúmenes, mientras la plataforma coordina los componentes que los proporcionan.

Proxmox también dispone de API, permisos y automatización. Por tanto, la comparación no consiste en atribuir esas capacidades exclusivamente a OpenStack, sino en entender cómo se articulan en cada plataforma. En OpenStack, la separación entre servicios forma parte de su arquitectura y también determina cómo se configura, se mantiene y se investiga un problema.

## 3. El mapa de servicios de OpenStack

| Servicio      | Responsabilidad                           | Referencia aproximada en Proxmox                           |
| ------------- | ----------------------------------------- | ---------------------------------------------------------- |
| **Nova**      | Computación y ciclo de vida de instancias | Gestión de máquinas virtuales                              |
| **Placement** | Inventarios y asignaciones de recursos    | Funciones de gestión de capacidad, sin equivalente directo |
| **Neutron**   | Redes virtuales                           | Networking y SDN                                           |
| **Glance**    | Catálogo y distribución de imágenes       | Imágenes y plantillas                                      |
| **Cinder**    | Volúmenes de bloques                      | Discos y almacenamiento                                    |
| **Swift**     | Almacenamiento de objetos                 | Sin equivalente nativo directo                             |
| **Keystone**  | Identidad y autorización                  | Usuarios, dominios y permisos                              |
| **Horizon**   | Interfaz web                              | Interfaz web de Proxmox                                    |
| **Octavia**   | Balanceadores de carga                    | Sin equivalente nativo directo                             |
| **Barbican**  | Secretos y claves                         | Sin equivalente nativo directo                             |
| **Heat**      | Orquestación de recursos                  | Sin equivalente nativo directo                             |

La tabla nos permite orientarnos, aunque no debemos buscar una correspondencia exacta entre ambas plataformas. Además, podemos empezar con los servicios que necesitamos e incorporar otros más adelante. Es lo que haremos en esta serie con Barbican y Octavia.

### 3.1. Una arquitectura basada en servicios

En la parte superior encontramos las herramientas de acceso: Horizon, CLI, SDK y clientes que consumen API. Por debajo, los servicios de OpenStack coordinan las solicitudes. Finalmente, tecnologías como KVM, Open vSwitch y Ceph ejecutan el trabajo real.

```text
┌────────────────────────────────────────────────────────────────────┐
│                  Usuarios · Horizon · CLI · API                    │
└────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌────────────────────────────────────────────────────────────────────┐
│                         Servicios OpenStack                        │
│                                                                    │
│  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐    │
│  │     Keystone     │ │       Nova       │ │     Neutron      │    │
│  └──────────────────┘ └──────────────────┘ └──────────────────┘    │
│                                                                    │
│  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐    │
│  │      Glance      │ │      Cinder      │ │    Placement     │    │
│  └──────────────────┘ └──────────────────┘ └──────────────────┘    │
└────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌────────────────────┐ ┌────────────────────┐ ┌────────────────────┐
│    KVM / libvirt   │ │      OVN / OVS     │ │        Ceph        │
└────────────────────┘ └────────────────────┘ └────────────────────┘
```

**OpenStack no reemplaza necesariamente las tecnologías de infraestructura: las coordina y las expone mediante servicios.**

## 4. Computación: Nova, Placement y KVM

**Nova** gestiona el ciclo de vida de las instancias: crear, arrancar, detener, reiniciar, redimensionar o eliminar. Pero Nova **no es el hipervisor**. En nuestro caso utilizará KVM a través de libvirt.

**Placement** mantiene información sobre inventarios y asignaciones de recursos. **Nova Scheduler** utiliza esa información y otros criterios de planificación para seleccionar un nodo adecuado. Finalmente, `nova-compute` coordina el arranque mediante libvirt y KVM.

Esta separación también nos será útil cuando algo falle. Si una instancia no encuentra un destino donde ejecutarse, tendremos que revisar la planificación y los recursos disponibles. Si el destino ya está seleccionado pero la máquina no arranca, seguiremos el recorrido hasta `nova-compute`, libvirt y KVM. Saber qué hace cada pieza nos da un lugar por el que empezar a investigar.

### Flavors: tamaños de instancia

Un *flavor* describe la capacidad de computación que solicita el usuario: vCPU, memoria y, según el caso, disco raíz o efímero. Podemos definir un `medium` de cuatro vCPU y 8 GB. El usuario elige el flavor sin preocuparse por las características de un servidor físico concreto.

## 5. Networking: Neutron y las redes virtuales

**Neutron** proporciona las API para administrar redes, subredes, puertos, routers y otros recursos de conectividad. Pensemos en dos proyectos que necesitan utilizar `192.168.10.0/24`. Gracias al aislamiento lógico, ambos pueden tener redes privadas con el mismo direccionamiento sin entrar en conflicto.

Los conceptos más importantes son:

- **Network:** red lógica donde conectamos instancias.
- **Subnet:** direccionamiento IP y parámetros asociados.
- **Port:** punto de conexión de red, normalmente asociado a una interfaz virtual.
- **Router:** conectividad de capa 3 entre redes y hacia el exterior.
- **Security Group:** reglas de filtrado de tráfico aplicadas a puertos.
- **Floating IP:** dirección de una red externa que puede asociarse a una instancia, normalmente mediante NAT.

### ¿Dónde encaja OVN?

Neutron define y administra los recursos, pero necesita tecnologías que implementen la conectividad. En nuestro laboratorio utilizaremos **OVN (Open Virtual Network)** y **Open vSwitch**. OVN proporciona la lógica de networking virtual; Open vSwitch participa en el forwarding de paquetes.

Más adelante profundizaremos en Geneve, gateways, bridges y conexión con redes físicas. Por ahora basta con recordar que **Neutron y OVN son complementarios, no equivalentes**.

## 6. Almacenamiento: Glance, Cinder, Swift y Ceph

Aquí merece la pena detenerse un momento. Si venimos de administrar discos y almacenamiento en Proxmox, es fácil intentar agrupar Glance, Cinder y Ceph bajo una misma etiqueta. Pero cada uno resuelve una parte diferente del problema.

Pensemos en una imagen de Ubuntu que vamos a utilizar para crear varias máquinas y en un disco con datos que queremos conservar aunque eliminemos una de ellas. Ambos necesitan almacenamiento, pero su función y su ciclo de vida son distintos.

### 6.1. Glance: el catálogo de imágenes

**Glance** administra el catálogo de imágenes utilizadas para crear instancias o volúmenes de arranque. Podemos publicar una imagen de Ubuntu 24.04, controlar su visibilidad y utilizarla como origen de nuevas máquinas. En Proxmox, el concepto recuerda al uso de plantillas o imágenes cloud, aunque el modelo de Glance es distinto.

### 6.2. Cinder: almacenamiento de bloques

**Cinder** ofrece volúmenes de bloques persistentes. Un usuario puede crear un volumen de 100 GB, conectarlo a una instancia y conservarlo aunque elimine la máquina, dependiendo de las políticas y opciones utilizadas. Cinder admite diferentes backends; nosotros emplearemos **Ceph RBD**.

### 6.3. Swift: almacenamiento de objetos

**Swift** proporciona almacenamiento de objetos mediante API. No presenta discos de bloques a las instancias. Su modelo es adecuado para aplicaciones que almacenan y recuperan objetos. No será una pieza central de nuestro primer despliegue.

### 6.4. Ceph: el almacenamiento subyacente

**Ceph no es un servicio de OpenStack.** Es una plataforma de almacenamiento distribuido con diferentes interfaces: RBD para bloques, CephFS para sistemas de archivos y RGW para objetos.

En nuestro laboratorio utilizaremos principalmente RBD. Glance podrá almacenar los datos de imágenes en Ceph; Cinder proporcionará volúmenes RBD; y Nova podrá utilizar Ceph para discos de instancias según la configuración elegida.

Cuando empecemos a configurar Ceph tendremos que decidir cómo replicar los datos y cómo distribuirlos entre los servidores. También comprobaremos qué ocurre con el quórum y el acceso al almacenamiento cuando falla un nodo. Tener tres servidores es el punto de partida; tendremos que demostrar qué fallos puede soportar esa configuración.

| Componente | Qué proporciona                       |
| ---------- | ------------------------------------- |
| Glance     | Catálogo y acceso a imágenes          |
| Cinder     | Volúmenes de bloques                  |
| Swift      | Objetos accesibles mediante API       |
| Ceph       | Almacenamiento distribuido subyacente |
| RBD        | Interfaz de bloques de Ceph           |

## 7. Identidad y servicios complementarios

**Keystone** gestiona identidad, autenticación, autorización y descubrimiento de servicios. Sus conceptos incluyen usuarios, grupos, dominios, proyectos y roles. Un proyecto delimita un ámbito de recursos y permisos: podríamos tener proyectos de desarrollo, preproducción y producción, cada uno con sus cuotas.

**Horizon** es la interfaz web tradicional. Permite gestionar instancias, redes, imágenes y volúmenes según los permisos del usuario. No es obligatorio utilizar Horizon: OpenStack puede administrarse por CLI, SDK y API.

Más adelante añadiremos **Octavia**, para ofrecer balanceadores de carga, y **Barbican**, para gestionar secretos y material criptográfico. Son buenos ejemplos de la modularidad de la plataforma.

## 8. ¿Qué ocurre realmente cuando creamos una máquina virtual?

La forma más sencilla de relacionar todas las piezas es seguir una solicitud. Un usuario quiere crear una instancia Ubuntu con cuatro vCPU, 8 GB de RAM y conexión a una red privada.

```text
┌─────────────────────────────────────────────────────────────────┐
│                          Horizon / CLI                          │
│                       Solicitud del usuario                     │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                            Keystone                             │
│                    Autenticación y autorización                 │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                            Nova API                             │
│                       Solicitud de creación                     │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Nova Scheduler + Placement                   │
│                       Selección de recursos                     │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                          Nova Compute                           │
│                    Coordinación de la instancia                 │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                          KVM / libvirt                          │
│                   Ejecución de la máquina virtual               │
└─────────────────────────────────────────────────────────────────┘

    Servicios que colaboran durante el aprovisionamiento:
┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐
│   Neutron / OVN   │ │       Glance      │ │   Cinder / Ceph   │
│   Red y puertos   │ │      Imágenes     │ │     Volúmenes     │
└───────────────────┘ └───────────────────┘ └───────────────────┘
```

**Paso 1. Autenticación.** El usuario se autentica mediante Keystone y trabaja dentro de un proyecto con permisos y cuotas determinados.

**Paso 2. Solicitud.** Desde Horizon o la CLI selecciona imagen, flavor y red. La petición llega a Nova API.

**Paso 3. Planificación.** Nova Scheduler utiliza Placement y otros criterios para elegir un nodo de cómputo adecuado.

**Paso 4. Networking.** Nova coordina con Neutron la preparación de los puertos. OVN y Open vSwitch implementan la conectividad virtual.

**Paso 5. Almacenamiento.** La imagen se obtiene a través de Glance. Si el arranque se realiza desde volumen, Cinder participa en su creación y presentación. Ceph RBD puede actuar como backend.

**Paso 6. Arranque.** `nova-compute` utiliza libvirt y KVM para configurar y ejecutar la máquina virtual.

**Paso 7. Acceso.** El usuario accede por la conectividad disponible. Si necesita exposición externa, podremos asociar una floating IP y ajustar las reglas de seguridad.

Desde la perspectiva del usuario, la operación parece sencilla. Internamente es una coordinación de servicios, procesos asíncronos y comprobaciones. Este recorrido simplifica algunos detalles, pero permite construir una primera imagen mental de OpenStack.

## 9. De los conceptos a nuestro laboratorio

En los siguientes artículos utilizaremos **tres nodos físicos y una máquina virtual bastión** para desplegar los componentes que acabamos de presentar. Este laboratorio nos permitirá acompañar cada explicación con su configuración y las comprobaciones correspondientes.

Veremos cómo distribuimos los servicios, qué redes utilizamos, cómo integramos Ceph y cómo desplegamos OpenStack con Kolla-Ansible. También dedicaremos una parte de la serie a la operación, las copias de seguridad y las pruebas de alta disponibilidad.

```text
┌──────────────────────────────────────────────────────────────────────┐
│                      BASTIÓN DE ADMINISTRACIÓN                       │
│     Ansible · Kolla · Registro Docker · Herramientas de operación    │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ↕
┌──────────────────────┐ ┌──────────────────────┐ ┌──────────────────────┐
│        NODO 1        │ │        NODO 2        │ │        NODO 3        │
├──────────────────────┤ ├──────────────────────┤ ├──────────────────────┤
│       Control        │ │       Control        │ │       Control        │
│       Compute        │ │       Compute        │ │       Compute        │
│       Ceph OSD       │ │       Ceph OSD       │ │       Ceph OSD       │
│     OVN Gateway      │ │     OVN Gateway      │ │     OVN Gateway      │
└──────────────────────┘ └──────────────────────┘ └──────────────────────┘
                                    │
                                    ↕
┌──────────────────────────────────────────────────────────────────────┐
│                       REDES REDUNDANTES Y VLAN                       │
│       Gestión · Ceph · Geneve · Provider · API pública               │
└──────────────────────────────────────────────────────────────────────┘
```

### La infraestructura

Los tres nodos serán equivalentes: un procesador físico, aproximadamente 64 GB de RAM por nodo, discos dedicados al sistema operativo y un SSD de unos 2 TB por nodo para Ceph. Partiremos de **Ubuntu 24.04** instalado y de conectividad redundante de 10 GbE con VLAN.

Usaremos una arquitectura **hiperconvergente**: los mismos nodos asumirán roles de control, computación y almacenamiento. Es un diseño adecuado para este laboratorio, aunque habrá que analizar sus implicaciones de capacidad, contención de recursos y dominios de fallo.

### Ceph como almacenamiento distribuido

Desplegaremos Ceph de forma independiente con `cephadm`, utilizando los tres SSD como OSD individuales. Configuraremos pools y permisos para integrarlo posteriormente con Glance, Cinder y Nova.

### Neutron y OVN para networking

Diseñaremos redes de gestión, almacenamiento, túneles Geneve, conectividad provider y acceso público. Los usuarios crearán redes privadas, routers y floating IPs. Separaremos el acceso de administración de los endpoints públicos.

### Kolla-Ansible para el despliegue

**Kolla** permite construir imágenes de contenedores de los servicios de OpenStack; **Kolla-Ansible** automatiza su despliegue y configuración mediante Ansible. Construiremos nuestras imágenes y las publicaremos en un Docker Registry privado ubicado inicialmente en el bastión.

Kolla-Ansible es una herramienta de despliegue y operación, **no un servicio de nube que consuman los usuarios**.

### Alta disponibilidad desde el diseño

Distribuiremos los servicios entre los tres nodos y validaremos sus mecanismos de redundancia. Conviene distinguir entre continuidad de las API, disponibilidad del almacenamiento y recuperación de máquinas virtuales: son problemas relacionados, pero no idénticos.

La prueba que interesa dejar para el final es bastante concreta: apagar uno de los servidores y comprobar qué sigue funcionando. ¿Podemos acceder a las API? ¿Qué ocurre con los volúmenes? ¿Y con las instancias que se estaban ejecutando en ese nodo? Tendremos que observar cada comportamiento por separado y contrastarlo con lo que esperábamos del diseño.

## 10. Conclusiones

La cantidad de nombres puede ser lo primero que nos frene al acercarnos a OpenStack, es un dolor de cabeza. Después de recorrerlos, ya podemos asociar cada uno con una responsabilidad: Nova gestiona las instancias, Placement mantiene el inventario y las asignaciones de recursos, Neutron administra las redes, Glance proporciona las imágenes y Cinder los volúmenes. Keystone nos permite identificar a los usuarios y trabajar dentro de sus proyectos y permisos.

Por debajo siguen estando tecnologías conocidas: KVM, Open vSwitch y Ceph. Ahora podemos empezar a seguir la relación entre lo que solicita un usuario y lo que termina ocurriendo en nuestros servidores.

Para mí, esa relación es lo que conviene tener claro antes de empezar con el despliegue. Nos ayudará tanto a preparar la configuración como a investigar los problemas que aparezcan después.

Queda por decidir cómo colocamos todas esas piezas en los tres nodos, qué redes utilizaremos y qué esperamos que ocurra si uno de los servidores falla.

**Ese será el siguiente artículo: diseñar nuestro clúster OpenStack hiperconvergente de tres nodos.** A partir de ese diseño empezaremos a preparar la infraestructura y a comprobar en el laboratorio si nuestras decisiones funcionan como esperamos.
