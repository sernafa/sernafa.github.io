---
layout: post
title: "Del clúster al servicio: Kubernetes sobre infraestructura propia"
date: 2020-03-20 11:00:00 +0100
categories: Kubernetes Networking
tags: kubernetes metallb traefik docker devops
description: "Instalación y configuración de Kubernetes On-Premise usando MetalLB, Traefik y diferentes recursos para desplegar y operar servicios."
---

# Del clúster al servicio: Kubernetes sobre infraestructura propia

En artículos anteriores hemos visto cómo realizar una instalación básica de Kubernetes, cómo desplegar MetalLB y algunos conceptos relacionados con `labels`, `matchLabels` y `selectors`.

En este artículo vamos a juntar todos estos elementos para pasar de tener un clúster funcionando a publicar un servicio dentro de nuestra infraestructura.

La instalación se ha realizado sobre máquinas virtuales con CentOS 7, Docker y Kubernetes 1.17. Para proporcionar una dirección IP a los servicios usaremos MetalLB y, para publicar las aplicaciones HTTP, Traefik como Ingress Controller.

> **Nota:** este artículo está fechado en marzo de 2020 y mantiene las versiones, comandos y API disponibles en ese momento. No debe utilizarse como procedimiento de instalación para versiones actuales de Kubernetes.

> Las salidas se han reconstruido y saneado a partir de las notas del laboratorio. Se han eliminado tokens, identificadores y otros datos que no deben publicarse.

## Entorno utilizado

El laboratorio está compuesto por tres máquinas virtuales:

| Servidor | Dirección IP | Función |
|---|---|---|
| `master` | `192.168.233.106` | Kubernetes Master |
| `worker1` | `192.168.233.107` | Kubernetes Worker |
| `worker2` | `192.168.233.108` | Kubernetes Worker |

Para la red de los POD usaremos el rango `10.244.0.0/16`. MetalLB utilizará las direcciones `192.168.233.200-192.168.233.210`, reservadas fuera del rango DHCP.

## Instalación básica del clúster

En todos los nodos tenemos que instalar Docker, `kubelet`, `kubeadm` y `kubectl`. También debemos deshabilitar el uso de swap.

```shell
# swapoff -a
# yum install -y yum-utils device-mapper-persistent-data lvm2
# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
# yum install -y docker-ce
# systemctl enable --now docker.service
```

Añadimos el repositorio de Kubernetes:

```text
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
```

Instalamos los componentes y preparamos la red:

```shell
# yum install -y kubelet kubeadm kubectl
# systemctl enable kubelet.service
# modprobe br_netfilter
# echo "net.bridge.bridge-nf-call-iptables=1" >> /etc/sysctl.conf
# sysctl -p
```

```text
net.bridge.bridge-nf-call-iptables = 1
```

En una instalación de laboratorio podemos desactivar temporalmente `firewalld` para simplificar las pruebas. En un entorno real deberíamos mantenerlo habilitado y abrir únicamente los puertos necesarios.

## Configuración del Master

Inicializamos el Master indicando el rango que utilizará Flannel para la red de los POD y la dirección IP del API Server.

```shell
# kubeadm init \
    --pod-network-cidr=10.244.0.0/16 \
    --apiserver-advertise-address=192.168.233.106
```

Salida abreviada:

```text
[init] Using Kubernetes version: v1.17.0
[preflight] Running pre-flight checks
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd
[kubelet-start] Starting the kubelet
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

kubeadm join 192.168.233.106:6443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

El token y el hash se han eliminado de la salida. No debemos guardar estos datos en el repositorio o publicarlos en un artículo.

Para trabajar con un usuario no privilegiado copiamos el fichero de configuración:

```shell
# useradd -m kube
# mkdir -p /home/kube/.kube
# cp -i /etc/kubernetes/admin.conf /home/kube/.kube/config
# chown -R kube:kube /home/kube/.kube
```

```shell
$ kubectl get nodes
```

```text
NAME     STATUS     ROLES    AGE   VERSION
master   NotReady   master   14m   v1.17.0
```

El estado aparece como `NotReady` porque todavía no hemos instalado el CNI que gestionará la red interna del clúster.

## Instalación de Flannel

En este laboratorio usaremos Flannel como CNI:

```shell
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

```text
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds-amd64 created
```

Esperamos unos segundos y volvemos a consultar el estado:

```shell
$ kubectl get nodes
```

```text
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    master   18m   v1.17.0
```

## Añadir los Workers

En cada Worker ejecutamos el comando generado por `kubeadm init`:

```shell
# kubeadm join 192.168.233.106:6443 --token <TOKEN> \
    --discovery-token-ca-cert-hash sha256:<HASH>
```

```text
[discovery] Trying to connect to API Server "192.168.233.106:6443"
[discovery] Cluster info signature and contents are valid
[kubelet-start] Activating the kubelet service
[tlsbootstrap] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.
```

Desde el Master comprobamos que ya tenemos los tres nodos:

```shell
$ kubectl get nodes -o wide
```

```text
NAME      STATUS   ROLES    AGE   VERSION   INTERNAL-IP       CONTAINER-RUNTIME
master    Ready    master   25m   v1.17.0   192.168.233.106   docker://19.3.5
worker1   Ready    <none>   7m    v1.17.0   192.168.233.107   docker://19.3.5
worker2   Ready    <none>   6m    v1.17.0   192.168.233.108   docker://19.3.5
```

## Desplegar un workload

Para probar el clúster vamos a desplegar un servidor Nginx. En lugar de crear un POD directamente usaremos un `Deployment`, que será el encargado de mantener el número de réplicas y realizar las actualizaciones.

```shell
$ kubectl create namespace web
```

Creamos `nginx-deployment.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: web
  labels:
    app: nginx
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      tier: frontend
  template:
    metadata:
      labels:
        app: nginx
        tier: frontend
    spec:
      containers:
        - name: nginx
          image: nginx:1.17.8
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 15
```

```shell
$ kubectl apply -f nginx-deployment.yml
$ kubectl get pods -n web -o wide
```

```text
deployment.apps/nginx created

NAME                     READY   STATUS    RESTARTS   AGE   IP           NODE
nginx-6fdb5d68b8-4j6jq   1/1     Running   0          35s   10.244.1.5   worker1
nginx-6fdb5d68b8-c8q7g   1/1     Running   0          35s   10.244.2.4   worker2
nginx-6fdb5d68b8-p5m2k   1/1     Running   0          35s   10.244.1.6   worker1
```

### Labels, matchLabels y selectors

El `matchLabels` del Deployment debe coincidir con los labels de la plantilla. De esta forma el Deployment sabe qué POD tiene que gestionar.

```shell
$ kubectl get pods -n web -l tier=frontend
```

```text
NAME                     READY   STATUS    RESTARTS   AGE
nginx-6fdb5d68b8-4j6jq   1/1     Running   0          2m
nginx-6fdb5d68b8-c8q7g   1/1     Running   0          2m
nginx-6fdb5d68b8-p5m2k   1/1     Running   0          2m
```

Creamos el Service interno usando el mismo selector:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: web
spec:
  type: ClusterIP
  selector:
    app: nginx
    tier: frontend
  ports:
    - name: http
      port: 80
      targetPort: 80
```

```shell
$ kubectl apply -f nginx-service.yml
$ kubectl get endpoints nginx -n web
```

```text
service/nginx created
NAME    ENDPOINTS                                    AGE
nginx   10.244.1.5:80,10.244.1.6:80,10.244.2.4:80   21s
```

Si `ENDPOINTS` aparece vacío, lo primero que debemos revisar son los labels de los POD y el selector del Service.

## Instalación de MetalLB

En Azure, AWS u otro Cloud Provider, un Service `LoadBalancer` solicita una IP al balanceador del proveedor. En una instalación On-Premise no tenemos este componente, por lo que instalaremos MetalLB 0.8.3.

```shell
$ kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.3/manifests/metallb.yaml
```

```text
namespace/metallb-system created
podsecuritypolicy.policy/speaker created
serviceaccount/controller created
serviceaccount/speaker created
clusterrole.rbac.authorization.k8s.io/metallb-system:controller created
clusterrole.rbac.authorization.k8s.io/metallb-system:speaker created
role.rbac.authorization.k8s.io/config-watcher created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:controller created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:speaker created
rolebinding.rbac.authorization.k8s.io/config-watcher created
daemonset.apps/speaker created
deployment.apps/controller created
```

Creamos el secreto necesario:

```shell
$ kubectl create secret generic -n metallb-system memberlist \
    --from-literal=secretkey="$(openssl rand -base64 128)"
```

Configuramos MetalLB en modo Layer 2:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
      - name: default
        protocol: layer2
        addresses:
          - 192.168.233.200-192.168.233.210
```

```shell
$ kubectl apply -f metallb-config.yml
$ kubectl get pods -n metallb-system
```

```text
configmap/config created
NAME                          READY   STATUS    RESTARTS   AGE
controller-65895b47d4-r9xgp   1/1     Running   0          61s
speaker-7ct2r                 1/1     Running   0          61s
speaker-f8mvk                 1/1     Running   0          61s
speaker-q4jx2                 1/1     Running   0          61s
```

El rango debe estar reservado y no puede ser entregado por DHCP. También podemos configurar MetalLB mediante BGP, estableciendo una sesión con nuestro router Mikrotik.

## Traefik como Ingress Controller

MetalLB proporciona una dirección IP, pero no decide a qué aplicación HTTP debe enviar cada petición. Para eso necesitamos un Ingress Controller.

Una vez desplegado Traefik, comprobamos que MetalLB le ha asignado una IP:

```shell
$ kubectl get svc -n traefik
```

```text
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)                      AGE
traefik   LoadBalancer   10.101.90.124   192.168.233.200   80:31680/TCP,443:31443/TCP   2m
```

Creamos una regla para publicar Nginx:

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: nginx
  namespace: web
spec:
  rules:
    - host: nginx.lab.local
      http:
        paths:
          - path: /
            backend:
              serviceName: nginx
              servicePort: 80
```

```shell
$ kubectl apply -f nginx-ingress.yml
$ kubectl get ingress -n web
```

```text
ingress.networking.k8s.io/nginx created
NAME    HOSTS             ADDRESS           PORTS   AGE
nginx   nginx.lab.local   192.168.233.200   80      14s
```

Añadimos `192.168.233.200 nginx.lab.local` al fichero `/etc/hosts` y probamos el acceso:

```shell
$ curl -I http://nginx.lab.local
```

```text
HTTP/1.1 200 OK
Content-Length: 612
Content-Type: text/html
Date: Fri, 20 Mar 2020 12:18:09 GMT
Server: nginx/1.17.8
```

El recorrido completo es:

```text
Cliente
  -> 192.168.233.200 (MetalLB)
  -> Traefik (Ingress Controller)
  -> Ingress nginx.lab.local
  -> Service nginx
  -> POD seleccionado mediante labels
```

## Observabilidad y comprobaciones básicas

Cuando el servicio deja de responder tenemos que revisar cada una de las capas anteriores.

```shell
$ kubectl rollout status deployment/nginx -n web
$ kubectl get events -n web --sort-by=.metadata.creationTimestamp
$ kubectl logs -n web nginx-6fdb5d68b8-4j6jq --tail=5
```

```text
deployment "nginx" successfully rolled out

LAST SEEN   TYPE     REASON      OBJECT                       MESSAGE
2m          Normal   Scheduled   pod/nginx-6fdb5d68b8-4j6jq   Successfully assigned web/nginx-6fdb5d68b8-4j6jq to worker1
2m          Normal   Started     pod/nginx-6fdb5d68b8-4j6jq   Started container nginx

10.244.0.5 - - [20/Mar/2020:12:18:09 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.29.0" "-"
```

Si tenemos Metrics Server también podemos consultar el consumo:

```shell
$ kubectl top nodes
$ kubectl top pods -n web
```

```text
NAME      CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
master    205m         5%     1238Mi          32%
worker1   96m          2%     742Mi           19%
worker2   89m          2%     705Mi           18%

NAME                     CPU(cores)   MEMORY(bytes)
nginx-6fdb5d68b8-4j6jq   1m           3Mi
nginx-6fdb5d68b8-c8q7g   1m           3Mi
nginx-6fdb5d68b8-p5m2k   1m           3Mi
```

Para disponer de históricos y alertas podemos instalar Prometheus y Grafana. Para centralizar logs podemos utilizar ELK, Graylog o una solución similar. Las sondas `readinessProbe` y `livenessProbe` permiten retirar del servicio un POD que no esté preparado o reiniciar un contenedor que haya dejado de responder.

## Consideraciones de seguridad

Aunque se trate de una instalación interna, debemos tener en cuenta algunas medidas básicas:

* No publicar el API Server directamente en Internet.
* No guardar tokens, certificados o ficheros `kubeconfig` en Git.
* Usar usuarios no privilegiados para la operación diaria.
* Aplicar RBAC y evitar asignar `cluster-admin` cuando no sea necesario.
* Separar las aplicaciones mediante namespaces.
* Definir límites de CPU y memoria.
* Utilizar imágenes con una versión determinada y evitar `latest`.
* Evitar contenedores privilegiados.
* Realizar copias de seguridad de etcd y probar su restauración.

Flannel proporciona conectividad entre POD, pero en su configuración básica no aplica `NetworkPolicy`. Si necesitamos controlar el tráfico entre aplicaciones tendremos que seleccionar un CNI que implemente estas políticas.

## Actualización y ciclo de vida

Antes de actualizar Kubernetes debemos revisar la compatibilidad de Docker, Flannel, MetalLB, Traefik y los manifiestos de las aplicaciones.

```shell
$ kubectl version --short
$ kubeadm version -o short
$ docker version --format '{{.Server.Version}}'
```

```text
Client Version: v1.17.0
Server Version: v1.17.0
v1.17.0
19.03.5
```

Para realizar mantenimiento sobre un Worker impedimos que reciba nuevos POD y evacuamos los existentes:

```shell
$ kubectl cordon worker1
$ kubectl drain worker1 --ignore-daemonsets --delete-local-data
```

```text
node/worker1 cordoned
evicting pod "nginx-6fdb5d68b8-4j6jq"
evicting pod "nginx-6fdb5d68b8-p5m2k"
pod/nginx-6fdb5d68b8-4j6jq evicted
pod/nginx-6fdb5d68b8-p5m2k evicted
node/worker1 evicted
```

Después del mantenimiento volvemos a habilitarlo:

```shell
$ kubectl uncordon worker1
```

```text
node/worker1 uncordoned
```

Debemos realizar el procedimiento nodo a nodo para mantener el servicio disponible. La copia de etcd contiene la configuración del clúster, pero no los datos almacenados en los volúmenes de las aplicaciones.

## Diferencias frente a Azure Kubernetes Service

La principal diferencia entre esta instalación y AKS es la responsabilidad sobre la infraestructura.

| Elemento | On-Premise | AKS |
|---|---|---|
| Master y etcd | Gestionado por nosotros | Gestionado por Azure |
| Workers | VM o servidores propios | Máquinas virtuales de Azure |
| LoadBalancer | MetalLB | Azure Load Balancer |
| Almacenamiento | Solución propia | Azure Disk y Azure Files |
| Identidad | Certificados y RBAC | Azure Active Directory y RBAC |
| Monitorización | Prometheus, Grafana, ELK, etc. | Azure Monitor o solución propia |
| Actualización | Todos los componentes | Azure gestiona el Master |

Para crear un clúster básico podemos utilizar Azure CLI:

```shell
$ az aks create \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --node-count 3 \
    --enable-addons monitoring \
    --generate-ssh-keys

$ az aks get-credentials \
    --resource-group myResourceGroup \
    --name myAKSCluster
```

```text
Merged "myAKSCluster" as current context in /home/kube/.kube/config
```

Azure gestiona el Master, pero seguimos siendo responsables de Deployments, Services, Ingress, permisos, imágenes, recursos, monitorización de las aplicaciones y actualización de los nodos.

## Conclusiones

Disponer de un clúster en estado `Ready` es sólo el primer paso. Para publicar una aplicación hemos configurado la red de los POD, desplegado el workload, relacionado el Deployment con un Service mediante labels y selectors, asignado una dirección IP con MetalLB y creado una regla de Ingress en Traefik.

También debemos añadir monitorización, logs, seguridad, copias de seguridad y un procedimiento de actualización. En una instalación On-Premise tenemos un mayor control sobre todos estos componentes, pero también asumimos toda su operación.

AKS simplifica parte de estas tareas al gestionar el plano de control e integrarse con otros servicios de Azure. La elección dependerá de la infraestructura disponible, los requisitos de integración y el nivel de control que necesitemos.

## Recursos

* [Kubernetes - Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
* [Kubernetes - Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
* [MetalLB](https://metallb.universe.tf/)
* [Traefik Kubernetes Ingress](https://docs.traefik.io/v2.1/providers/kubernetes-ingress/)
* [Azure Kubernetes Service](https://docs.microsoft.com/es-es/azure/aks/)
