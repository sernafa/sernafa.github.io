---
layout: post
title: "De Docker Compose a GitOps distribuido: gestionando proxies de Zabbix remotos con Argo CD"
date: 2026-09-13 20:14:00 +0200
categories: [Homelab, GitOps]
tags: [gitops, zabbix, argocd, kubernetes, k3s, docker]
toc: true
---

Hay un momento en el que una solución técnica perfectamente válida
empieza a generar un problema operativo.

No porque la tecnología sea incorrecta.

No porque necesariamente haya que sustituirla.

Simplemente porque el entorno alrededor de ella ha crecido.

Este artículo nace precisamente de una de esas situaciones.

Trabajo con entornos Zabbix distribuidos donde existen proxies
desplegados en numerosas ubicaciones remotas. Podemos estar hablando de
decenas de proxies ---del orden de 50 instalaciones--- distribuidos en
entornos con diferentes mecanismos de acceso, restricciones de red y
particularidades propias.

Además, partimos de una situación bastante razonable: **los proxies ya
están contenerizados**.

No estamos hablando de migrar antiguas instalaciones basadas en paquetes
o binarios hacia contenedores, ni este artículo pretende ser otro
ejercicio de *"vamos a meter Kubernetes en todo"*.

El modelo actual es bastante sencillo:

![Despliegue de Zabbix Proxy: ubicación remota → máquina virtual → Docker Compose → Zabbix Proxy.](/assets/img/posts/gitops-distribuido-zabbix-argocd/zabbix-proxy-docker-compose.png){: width="512" height="768" }

Y funciona.

El problema aparece cuando empezamos a analizar el **ciclo de vida
operativo** de todos esos despliegues.

Cuando actualizamos la plataforma central de Zabbix, tarde o temprano
necesitamos actualizar también los proxies.

Con tres o cuatro proxies, realizar esa actualización manualmente es
perfectamente asumible.

Con 30, 40 o 50 instalaciones distribuidas entre diferentes ubicaciones
o clientes, la situación cambia.

Tenemos que acceder a cada entorno, autenticarnos mediante el mecanismo
disponible en cada caso, actualizar la imagen del contenedor,
desplegarla y comprobar posteriormente que todo sigue funcionando
correctamente.

Y repetir el proceso.

Una vez.

Y otra.

Y otra.

En ese momento, el problema ya no es Docker Compose.

El problema es **cómo gestionar el ciclo de vida de una plataforma
distribuida**.

Y eso me llevó a plantearme una pregunta:

> Si las cargas ya están contenerizadas, ¿podríamos utilizar GitOps para
> gestionar su ciclo de vida sin necesidad de acceder individualmente a
> cada sistema remoto?

Eso es precisamente lo que quiero explorar.

------------------------------------------------------------------------

## En realidad, este artículo no trata sobre Zabbix

Zabbix es el caso de uso que desencadenó la idea, pero creo que el
patrón es bastante más amplio.

Imaginemos un equipo de servicios gestionados encargado de operar
software desplegado en las infraestructuras de múltiples clientes.

Cada cliente tiene sus propios límites de seguridad, infraestructura,
conectividad y políticas de acceso.

Al mismo tiempo, queremos que un equipo centralizado pueda controlar el
ciclo de vida del software, manteniendo un aislamiento claro entre
clientes.

Y existe otro requisito que considero especialmente importante:

**los entornos remotos deberían necesitar únicamente conectividad
saliente.**

No quiero que nuestra plataforma central necesite acceder directamente a
la API de Kubernetes de cada cliente.

Eso implicaría empezar a introducir VPN, reglas de firewall, routing,
APIs expuestas o diferentes mecanismos de acceso remoto.

Precisamente una de las cosas que queremos evitar.

La arquitectura que tengo en mente se parece más a esto:

![Arquitectura GitOps distribuida: los clientes A, B y N ejecutan K3s, argocd-agent y Zabbix Proxy e inician conexiones salientes hacia el plano de gestión con Argo CD y argocd-agent principal.](/assets/img/posts/gitops-distribuido-zabbix-argocd/arquitectura-gitops-distribuido.png)

Es el entorno remoto quien inicia la conexión.

El plano de gestión determina el estado deseado.

Y Git se convierte en el lugar donde describimos ese estado.

------------------------------------------------------------------------

# ¿Por qué Kubernetes?

No creo que añadir Kubernetes convierta automáticamente una arquitectura
en algo mejor.

Nuestro punto de partida con Docker Compose ya funciona.

Por tanto, la pregunta no debería ser:

> ¿Cómo podemos sustituir Docker Compose por Kubernetes?

Creo que existe una pregunta bastante más interesante:

> ¿Puede una capa ligera de Kubernetes proporcionarnos las primitivas
> necesarias para gestionar estas cargas distribuidas de forma
> declarativa?

Para este experimento he elegido **K3s**.

El workload de un Zabbix Proxy no requiere desplegar una gran plataforma
Kubernetes en cada ubicación remota.

Lo que buscamos principalmente es disponer de un entorno de ejecución
Kubernetes ligero sobre el que podamos aplicar un modelo GitOps.

Y una vez disponemos de Kubernetes, Argo CD se convierte en un candidato
bastante interesante para atacar nuestro problema original.

------------------------------------------------------------------------

# Git como interfaz de operación

Supongamos que tenemos un cliente llamado `customer-a`.

Actualmente una actualización podría requerir conectarnos al servidor y
ejecutar algo equivalente a:

``` bash
docker compose pull
docker compose up -d
```

Lo que me gustaría conseguir es que el procedimiento operativo fuese
algo parecido a:

![Flujo de actualización GitOps: cambiar versión de imagen → commit → push → Argo CD detecta el cambio → el clúster remoto reconcilia → Zabbix Proxy actualizado.](/assets/img/posts/gitops-distribuido-zabbix-argocd/flujo-actualizacion-gitops.png){: width="512" height="768" }

Sin SSH.

Sin ejecutar comandos manualmente sobre el nodo remoto.

Sin necesidad de que un técnico acceda a la infraestructura del cliente.

**Git pasa a convertirse en nuestra interfaz de operación.**

Además, obtenemos algo igualmente importante: trazabilidad.

Podemos responder fácilmente a preguntas como:

-   ¿Quién cambió la versión?
-   ¿Cuándo?
-   ¿De qué versión veníamos?
-   ¿A qué versión hemos actualizado?
-   ¿Por qué se realizó el cambio?

Y si necesitamos volver atrás, el rollback vuelve a ser simplemente otro
cambio en Git.

------------------------------------------------------------------------

# Un repositorio por cliente

Una de las decisiones arquitectónicas que quería tomar desde el
principio era cómo organizar los repositorios.

Podríamos utilizar un único repositorio con una estructura similar a:

``` text
zabbix-proxies/
├── customer-a/
├── customer-b/
├── customer-c/
└── customer-d/
```

Sin embargo, quiero explorar deliberadamente otro modelo.

**Cada cliente tendrá su propio repositorio.**

Por ejemplo:

``` text
zabbix-proxy-template

zabbix-proxy-customer-a
zabbix-proxy-customer-b
zabbix-proxy-customer-c
...
```

El primer repositorio representa nuestra plantilla común.

Cuando damos de alta un nuevo cliente, utilizamos esa plantilla como
punto de partida y creamos un repositorio independiente para él.

La imagen del Zabbix Proxy será común.

Lo que cambiará será la configuración.

Esto nos proporciona una característica que me parece especialmente
interesante: **cada cliente tiene su propio ciclo de vida**.

Imaginemos que todos están utilizando la misma versión.

Podemos comenzar el proceso de actualización únicamente con el cliente
A:

``` yaml
image:
  tag: 7.x.z
```

El cliente B puede continuar temporalmente en la versión anterior.

El C puede actualizarse después.

Y quizás el D tenga algún requisito particular que haga conveniente
retrasar su actualización.

Además, reducimos el *blast radius* de cada cambio.

Un commit en el repositorio del cliente A no debería modificar
absolutamente nada en el cliente B.

------------------------------------------------------------------------

# GitOps no significa eliminar la decisión humana

No quiero construir un mecanismo que automáticamente actualice todos los
proxies en cuanto aparezca una nueva versión de Zabbix.

Existe un equipo humano responsable del ciclo de vida de la plataforma.

Ese equipo decide **qué versión desplegar, cuándo desplegarla y sobre
qué clientes hacerlo**.

GitOps no sustituye esa decisión.

Lo que hace es ejecutar de forma consistente el estado que ese equipo ha
decidido.

La automatización no tiene por qué significar ausencia de control
humano.

En este caso quiero precisamente lo contrario: **automatizar la
ejecución manteniendo humana la decisión**.

------------------------------------------------------------------------

# Una imagen personalizada de Zabbix Proxy

Otra pieza del experimento será el propio contenedor.

La idea es mantener **una única imagen personalizada**, derivada de la
imagen oficial de Zabbix Proxy.

Conceptualmente:

``` dockerfile
FROM zabbix/zabbix-proxy-sqlite3:<version>

# Herramientas adicionales
# Certificados personalizados
# Scripts operativos
# Utilidades de troubleshooting
# etc.
```

No queremos construir una imagen diferente para cada cliente.

Las diferencias deberían estar en la configuración.

El ciclo de vida sería aproximadamente:

![Flujo de distribución: imagen oficial Zabbix Proxy → GitHub Actions → imagen personalizada → GHCR → repositorios Git de clientes → Argo CD → clústeres remotos.](/assets/img/posts/gitops-distribuido-zabbix-argocd/flujo-imagen-personalizada-zabbix.png){: width="512" height="768" }

Inicialmente tampoco veo necesario automatizar completamente esta parte.

Un workflow de GitHub Actions ejecutado mediante `workflow_dispatch`
sería suficiente.

El operador selecciona la versión upstream de Zabbix, GitHub Actions
construye nuestra imagen personalizada y la publica en GHCR.

En el futuro podríamos detectar automáticamente nuevas versiones
upstream.

Pero para demostrar el concepto no necesitamos llegar hasta ahí.

Automatizar algo simplemente porque podemos automatizarlo tampoco
debería ser el objetivo.

------------------------------------------------------------------------

# El problema interesante: la conectividad

El modelo multi-clúster tradicional de Argo CD permite gestionar
múltiples clústeres Kubernetes desde una instalación central.

Pero existe una consecuencia.

El Argo CD central necesita conectividad con las APIs Kubernetes de esos
clústeres remotos.

Y eso es precisamente algo que quiero evitar.

El requisito es:

> El cliente debe poder iniciar conexiones hacia nuestra plataforma
> central, pero nuestra plataforma central no debería necesitar iniciar
> conexiones hacia la infraestructura del cliente.

Aquí es donde entra en juego **argocd-agent**.

El proyecto introduce una arquitectura basada en dos elementos:

``` text
principal  <──────  agent
```

El `principal` reside en el plano de gestión.

El `agent` reside en el clúster remoto.

En el modelo *managed*, las Applications pueden definirse desde el plano
central y distribuirse al agente correspondiente.

El agente trabaja localmente con los componentes necesarios de Argo CD y
posteriormente devuelve el estado al principal.

Y, sobre todo, hay una característica que encaja especialmente bien con
nuestro requisito:

**la conexión se inicia desde el agente hacia el principal.**

Nuestra arquitectura empieza entonces a tomar esta forma:

![El Agent del cliente remoto inicia una conexión saliente mTLS hacia Argo CD y Principal en el plano central. El cliente incluye Application Ctrl., Repo Server y Redis, con el flujo local hacia Kubernetes y Zabbix Proxy.](/assets/img/posts/gitops-distribuido-zabbix-argocd/conexion-mtls-cliente-remoto.png){: width="512" height="768" }

Esto se aproxima mucho más al modelo operacional que queremos conseguir.

------------------------------------------------------------------------

# Un cliente, un proyecto, un destino

En el plano central podemos representar cada cliente de forma
independiente.

Conceptualmente podríamos terminar teniendo:

![Argo CD organiza una Application por cliente: Customer A usa el repositorio zabbix-proxy-customer-a y el destino customer-a; Customer B usa zabbix-proxy-customer-b y customer-b; Customer C usa zabbix-proxy-customer-c y customer-c.](/assets/img/posts/gitops-distribuido-zabbix-argocd/argocd-applications-clientes.png){: width="1536" height="1024" }

De esta forma mantenemos separadas varias responsabilidades.

La imagen define **qué ejecutamos**.

El repositorio del cliente define **cómo queremos ejecutarlo en ese
cliente**.

Argo CD define **dónde debe ejecutarse y cuál debe ser su estado**.

Y el agente permite llevar esa intención hasta el clúster remoto.

------------------------------------------------------------------------

# Construyendo un pequeño laboratorio

Todo el laboratorio será sintético.

Para ello podemos utilizar **K3d**, lo que nos permite levantar varios
clústeres K3s localmente.

Nuestro pequeño universo tendrá tres clústeres:

![Laboratorio GitOps: k3d-central contiene Argo CD y Principal. Los clústeres k3d-customer-a y k3d-customer-b contienen Agent, App Controller, Repo Server, Redis y Zabbix Proxy, y se conectan hacia el clúster central.](/assets/img/posts/gitops-distribuido-zabbix-argocd/laboratorio-k3d-gitops.png){: width="1536" height="1024" }

No pretende ser una reproducción de producción.

Tampoco pretende demostrar cómo debemos desplegar K3s en un cliente.

De hecho, establecemos una frontera clara:

> **El aprovisionamiento de la máquina virtual y la instalación inicial
> de K3s quedan fuera del alcance de esta PoC.**

Partimos de que ese K3s ya existe.

Automatizar su creación podría ser perfectamente otro proyecto
utilizando Terraform, Ansible o herramientas similares.

Pero ese es otro problema.

Y prefiero no intentar resolver diez problemas diferentes dentro del
mismo artículo.

------------------------------------------------------------------------

# La PoC en cuatro actos

## Acto 1 --- Levantar el plano de control

Creamos:

``` text
k3d-central
```

Instalamos Argo CD y el componente principal de `argocd-agent`.

Todavía no tenemos ningún workload de cliente.

Simplemente queremos comprobar que nuestro plano de gestión funciona
correctamente.

## Acto 2 --- Conectar los clientes

Creamos:

``` text
k3d-customer-a
k3d-customer-b
```

Ambos representan instalaciones K3s que ya existirían en clientes
reales.

Instalamos los componentes necesarios del lado del agente y comprobamos
que son capaces de conectarse al principal utilizando conexiones
iniciadas desde los clústeres remotos.

## Acto 3 --- Desplegar los proxies

Creamos dos Applications.

Una referencia:

``` text
zabbix-proxy-customer-a
```

La otra:

``` text
zabbix-proxy-customer-b
```

Ambos clientes comienzan utilizando la misma versión:

``` text
customer-a → proxy:v1
customer-b → proxy:v1
```

Esperamos hasta obtener:

``` text
customer-a → Synced / Healthy
customer-b → Synced / Healthy
```

Ese será nuestro punto de partida.

## Acto 4 --- La prueba que realmente importa

Entramos únicamente en:

``` text
zabbix-proxy-customer-a
```

Y modificamos:

``` diff
- image: proxy:v1
+ image: proxy:v2
```

Después:

``` bash
git commit
git push
```

Y nada más.

No accedemos por SSH.

No ejecutamos `kubectl` contra el clúster remoto para realizar el
upgrade.

No hacemos un `docker pull`.

No reiniciamos manualmente ningún servicio.

Argo CD detecta que el estado deseado ha cambiado.

La Application llega al agente correspondiente.

El controlador local reconcilia el estado.

Kubernetes realiza la actualización.

Y finalmente deberíamos tener:

``` text
customer-a → proxy:v2
customer-b → proxy:v1
```

Ese pequeño resultado es, en realidad, **la demostración completa de la
idea**.

------------------------------------------------------------------------

# El rollback debería ser aburrido

El rollback no debería ser una operación especial.

Simplemente hacemos:

``` diff
- image: proxy:v2
+ image: proxy:v1
```

Commit.

Push.

Y dejamos que la plataforma vuelva a reconciliar.

Si cuando aparece un problema necesitamos abandonar GitOps, conectarnos
manualmente al servidor y empezar a ejecutar comandos, entonces
probablemente no hemos solucionado realmente nuestro problema inicial.

------------------------------------------------------------------------

# ¿Cuándo consideraré que la PoC funciona?

Para mí, el criterio de éxito no es:

> Hemos conseguido instalar Argo CD.

La pregunta que quiero responder es otra:

> **¿Puedo actualizar un Zabbix Proxy remoto modificando únicamente Git,
> permitir que la plataforma reconcilie el cambio, verificarlo desde el
> plano central y posteriormente revertirlo sin acceder manualmente al
> nodo remoto?**

Además, quiero comprobar que puedo hacerlo sobre un cliente sin afectar
a los demás.

Si podemos demostrar esas dos cosas, la arquitectura merece seguir
siendo explorada.

Y si descubrimos que no funciona como esperamos, también habremos
aprendido algo.

Al fin y al cabo, para eso sirve una PoC.

------------------------------------------------------------------------

# Lo que deliberadamente no estamos resolviendo

Hay varias cuestiones importantes que no quiero esconder debajo de la
alfombra.

La primera es probablemente la más evidente:

**la gestión de secretos.**

Un despliegue real de Zabbix Proxy puede necesitar una identidad PSK y
su correspondiente material criptográfico.

Meter esos valores directamente en un repositorio Git convencional no es
una solución.

Existen diferentes alternativas que podríamos explorar:

-   External Secrets Operator junto con un secret store externo.
-   SOPS para almacenar secretos cifrados en Git.
-   Sealed Secrets.
-   HashiCorp Vault.
-   Otros sistemas centralizados de gestión de secretos.

No quiero elegir uno simplemente para completar el diagrama.

Creo que esa decisión merece su propio análisis.

También quedan otras preguntas interesantes: cómo propagamos cambios
realizados en `zabbix-proxy-template` hacia repositorios existentes,
cómo gestionamos actualizaciones progresivas con decenas de clientes,
cómo detenemos una promoción ante un problema, qué observabilidad
necesitamos sobre los agentes, qué ocurre cuando un cliente permanece
desconectado durante horas, qué disponibilidad necesitamos en el plano
central y cómo automatizamos el bootstrap completo de un nuevo cliente.

Todas esas cuestiones son relevantes.

Pero ninguna es necesaria para responder a nuestra pregunta inicial.

------------------------------------------------------------------------

# Lo que realmente me interesa de este enfoque

Lo que más me atrae de esta arquitectura no es Kubernetes.

Tampoco Argo CD.

Ni siquiera Zabbix.

Lo interesante es el cambio en el **modelo operacional**.

Pasamos de algo parecido a:

![Operación manual: el técnico conecta con el Cliente A para actualizar y verificar, repite ambas tareas con el Cliente B y continúa con los demás clientes.](/assets/img/posts/gitops-distribuido-zabbix-argocd/operacion-manual-clientes.png){: width="512" height="768" }

a:

![El equipo de operación define en Git el estado deseado; la reconciliación GitOps lo aplica a los clientes A, B, C y demás destinos.](/assets/img/posts/gitops-distribuido-zabbix-argocd/operacion-gitops-clientes.png){: width="512" height="768" }

El equipo humano sigue tomando las decisiones importantes.

¿Qué versión queremos desplegar?

¿Qué cliente actualizamos primero?

¿Continuamos con la siguiente tanda?

¿Detenemos el rollout?

¿Volvemos atrás?

La plataforma no debería decidir eso por nosotros.

La plataforma debería hacer que **nuestras decisiones sean
reproducibles, trazables y consistentes**.

------------------------------------------------------------------------

# ¿Estamos complicando algo que ya funcionaba?

También creo que hay que hacerse esta pregunta.

Porque la respuesta puede ser perfectamente **sí**.

Si tenemos tres proxies, probablemente Docker Compose y algo de
automatización sean más que suficientes.

Introducir K3s, Argo CD, agentes, repositorios Git y todo el ecosistema
asociado tendría un coste que difícilmente justificaríamos.

Pero cuando pasamos de tres instalaciones a varias decenas, la ecuación
empieza a cambiar.

Especialmente cuando añadimos:

![Factores que se suman a la complejidad operativa: número de instalaciones, diferentes métodos de acceso, restricciones de red, equipos mutualizados, necesidad de trazabilidad y ciclos de actualización frecuentes.](/assets/img/posts/gitops-distribuido-zabbix-argocd/factores-complejidad-operativa.png){: width="512" height="768" }

En ese escenario merece la pena, como mínimo, hacer el experimento.

No para demostrar que Kubernetes es la solución.

Sino para comprobar si **este modelo operacional** es mejor que el que
tenemos actualmente.

------------------------------------------------------------------------

# Conclusiones

Todo esto empezó con un problema bastante cotidiano.

Actualizar muchos Zabbix Proxies remotos es tedioso.

Pero al analizar ese problema aparece una cuestión bastante más
interesante: **cómo operar software distribuido cuando un mismo equipo
gestiona múltiples entornos remotos y aislados entre sí**.

Nuestro punto de partida ya era razonablemente moderno.

Los servicios estaban contenerizados.

Docker Compose funcionaba.

Por tanto, no necesitábamos reinventar la plataforma.

Lo que necesitábamos cuestionarnos era su operación.

Y ahí es donde K3s, GitOps, Argo CD y `argocd-agent` empiezan a resultar
interesantes.

La hipótesis es relativamente sencilla:

![Ciclo GitOps con control humano: humano decide → Git describe → Argo distribuye → Kubernetes reconcilia → humano verifica.](/assets/img/posts/gitops-distribuido-zabbix-argocd/ciclo-gitops-control-humano.png){: width="512" height="768" }

Todavía quedan muchas preguntas.

Secretos.

Bootstrap.

Observabilidad.

Rollouts masivos.

Propagación de templates.

Alta disponibilidad.

Seguramente aparecerán algunas más cuando empecemos a romper cosas en el
laboratorio.

Y precisamente esa es la siguiente parte interesante.

Convertir esta arquitectura en algo reproducible, ejecutar la PoC y
comprobar dónde nuestras suposiciones funcionan...

...y dónde no.

Porque, personalmente, esa es una de las partes que más me interesan
cuando exploro una tecnología nueva:

**partir de un problema real, encontrar una tecnología que parece que
podría encajar y comprobar de verdad si lo hace.**

No Kubernetes porque Kubernetes.

No GitOps porque GitOps.

Simplemente utilizar otra herramienta cuando el problema hace que tenga
sentido.
