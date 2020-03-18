---
title:      "Kubernetes Cluster, instalación básica"
date:       2020-02-25 11:00:04
excerpt:    "Proceso de instalación de un cluster Kubernetes On-Prime"
tags:       kubernetes cluster
header:
    teaser: /assets/images/kubernetes_teaser.png
---

## Instalación, configuración básica de los nodos del cluster

En cada uno de los nodos que queramos que forme parte del cluster de *Kubernetes*, tendremos que realizar estos pasos básicos de instalación y configuración.

### Cambios en el sistema base

La instalación del cluster Kubernetes *on-premise* lo he realizado usando de base *CentOS 7* en VM con VMware 6.7, con el último nivel de parches.  Los requerimientos mínimos de las VM para el manager / worker son:

* 2 vCPU y 2 GB de RAM
* IP estatica en cada nodo
* Deshabilitar cualquier *swap* del sistema.

En *CentOS 7* lo primero es desactivar, para evitar ningún problema, *SELinux y *FirewallD*, ademas de incluir flags en el *sysctl.conf* del sistema, y reiniciamos.

```shell
# sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
# systemctl disable firewalld
# echo "net.bridge.bridge-nf-call-iptables=1" >> /etc/sysctl.conf
# reboot
```

### Instalacion Docker

Todos los nodos necesitarán tener instalado Docker, lo conseguimos con los siguientes pasos:

```shell
# yum install -y yum-utils device-mapper-persistent-data lvm2
# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
# yum install -y docker-ce
# systemctl enable docker.service
# systemctl start docker.service
```

### Instalación Kubernetes

Por último añadimos *Kubernetes* a cada uno de los nodos, incluyendo un nuevo repositorio e instalado las piezas necesarias:

```shell
# vi /etc/yum.repos.d/kubernetes.repo
```

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

```shell
# yum install -y kubelet kubeadm kubectl
# systemctl enable kubelet.service
```

## MANAGER, configuración

Levantamos el servicio de manager del cluster.

```shell
# kubeadm init  --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.233.106

....

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

kubeadm join 192.168.233.106:6443 --token fwldpz.bgejrgr1cqviuoeo --discovery-token-ca-cert-hash sha256:b11fbe1063550e822c54033729d45ca28e1fdb05deed3fbd103f27cf2926b2e7
```

Para la gestión, se recomienda el uso de un usuario no privilegiado, se crea y le se copia el fichero de configuración/acceo al cluster.

```shell
# useradd -m kube
# mkdir -p /home/kube/.kube
# cp -i /etc/kubernetes/admin.conf /home/kube/.kube/config
# chown kube:kube /home/kube/.kube/config
```

Y comprobamos el funcionamiento del manager del cluster.

```shell
$ kubectl get nodes

NAME          STATUS     ROLES    AGE   VERSION
master         NotReady   master   14m   v1.17.0
```

En la salida del mensaje anterior nos muestra que el nodo master se encuentra en el estado NotReady, esto es por que nos falta instalar un CNI (Container Networking Interface).  En este caso instalaremos [Flannel](https://coreos.com/flannel/docs/latest/kubernetes.html) como nuestro gestor de red.  Aplicamos al cluster:

```shell
$ kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml
```

```shell
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                        READY   STATUS     RESTARTS   AGE
kube-system      coredns-6955765f44-rqtxz                    1/1     Running   0          19h
kube-system      coredns-6955765f44-s8ztq                    1/1     Running   0          19h
kube-system      etcd-cos-kstage-master                      1/1     Running   0          19h
kube-system      kube-apiserver-cos-kstage-master            1/1     Running   0          19h
kube-system      kube-controller-manager-cos-kstage-master   1/1     Running   0          19h
kube-system      kube-flannel-ds-amd64-8msj8                 1/1     Running   1          19h
kube-system      kube-flannel-ds-amd64-twpz5                 1/1     Running   0          19h
kube-system      kube-flannel-ds-amd64-wc8jg                 1/1     Running   0          18h
kube-system      kube-proxy-nlfhk                            1/1     Running   0          19h
kube-system      kube-proxy-pxclb                            1/1     Running   0          18h
kube-system      kube-proxy-zh4gc                            1/1     Running   1          19h
kube-system      kube-scheduler-cos-kstage-master            1/1     Running   0          19h
```

Una vez que todos los pods tengan el estado RUNNING ejecutamos el siguiente comando:

```shell
$ kubectl get nodes

NAME          STATUS     ROLES    AGE     VERSION
master         Ready    master   6m44s   v1.17.0
```

## WORKERS, configuración

En los workers solo debemos de ejecutar el comando que nos ofrecio el *manager*:

```
# kubeadm join 192.168.233.106:6443 --token fwldpz.bgejrgr1cqviuoeo --discovery-token-ca-cert-hash sha256:b11fbe1063550e822c54033729d45ca28e1fdb05deed3fbd103f27cf2926b2e7

discovery] Trying to connect to API Server "192.168.233.106:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://192.168.233.106:6443"
[discovery] Requesting info from "https://192.168.233.106:6443" again to validate TLS against the pinned public key
[discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "192.168.1.105:6443"
[discovery] Successfully established connection with API Server "192.168.233.106:6443"
[join] Reading configuration from the cluster...
[join] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet] Downloading configuration for the kubelet from the "kubelet-config-1.13" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[tlsbootstrap] Waiting for the kubelet to perform the TLS Bootstrap...
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "worker-node" as an annotation
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.
Run 'kubectl get nodes' on the master to see this node join the cluster.
```

Ahora podemos volver a preguntar por estado de los nodos.

```shell
$ kubectl get nodes

NAME          STATUS     ROLES    AGE     VERSION
master         Ready    master   6m44s   v1.17.0
worker1        Ready    <none>   5m29s   v1.17.0
```

## Kubernetes Cluster, desplegando el primer POD

Creamos un workload y servicio *Nginx*.

```shell
$ kubectl create deployment nginx --image=nginx
deployment.apps/nginx created
```

Ahora creamos su servicio asociado para poder acceder al *Nginx*.

```shell
$ kubectl create service nodeport nginx --tcp=80:80
```

Consultamos el servicio para saber el *NodePort* de acceso.

```shell
$ kubectl get svc

NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx       NodePort    10.102.166.47   <none>        80:30784/TCP   31s
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        25m
```

Lanzamos un *CURL* a la IP de cualquiera de los *workers*:

```shell
$ curl http://192.168.1.106:30784

$ curl http://192.168.233.107:30345
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## Links

[https://medium.com/liveness-y-readiness-probe/instalaci%C3%B3n-de-kubernetes-onpremise-638609f2bb1e](https://medium.com/liveness-y-readiness-probe/instalaci%C3%B3n-de-kubernetes-onpremise-638609f2bb1e)