# ArgoCD en Kubernetes con kind

## Introducción

En este guía mostramos como instalar un cluster de `Kubernetes` en la laptop usando la implementación
de `kind`, la cual corre cada componente en un contenedor en lugar de usar máquinas virtuales, originalmente
fue diseñado para probar kubernetes en sí, pero también puede ser usado para desarrollo local o CI.

Este proyecto puede servir para comprender los conceptos, la arquitectura y adentrarnos más en lo que
son los contenedores, los pods y su relación con los micro servicios.

Instalaremos `Kong` como Ingress Controller para exponer algunos servicios web que desplegaremos.

Instalaremos `ArgoCD` para realizar los procesos de despliegues de los diferentes servicios y
aplicaciones en el cluster Kubernetes.

## Requisitos

Es necesario tener instalado y en ejecución docker para poder gestionar contenedores, este ejercicio lo
realizaremos en un equipo con MacOS, por lo que instalaremos la implementación `colima` para correr docker
en local, si tienes Linux puedes instalar docker usando tu manejador de paquetes favorito.

Iniciamos instalando colima y el cliente docker:

```shell
$ brew install colima docker
```

Ahora debemos iniciar colima:

```shell
$ colima start
INFO[0000] starting colima
INFO[0000] runtime: docker
INFO[0001] creating and starting ...           context=vm
INFO[0044] provisioning ...                    context=docker
INFO[0044] starting ...                        context=docker
INFO[0045] done
```

**NOTA:** Por default colima levanta una máquina virtual con `2` vCPUs y `2` GB de RAM, si se desea modificar
esto para asignar más CPU o RAM, puedes agregar los parámetros `--cpu 4` y `--memory 4`.

Ahora instalamos los paquetes para kubernetes con `kind`, también instalamos el cliente `kubectl` y
`k6` la herramienta de pruebas de carga de aplicaciones web:

```shell
$ brew install kind kubectl helm k6
```

Validamos la instalación de las herramientas, iniciamos con kind:

```shell
$ kind --version
kind version 0.22.0
```

Ahora veamos la versión de `kubectl`:

```shell
$ kubectl version --client=true
Client Version: v1.30.0
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
```

Veamos la versión de `helm`:

```shell
$ helm version
version.BuildInfo{Version:"v3.13.0"}
```

Y finalmente la versión de `k6`:

```shell
$ k6 version
k6 v0.46.0 ((devel))
```

## Instalación de cluster

Definimos la configuración del cluster con dos nodos, uno con rol de `control-plane` y otros dos de `worker`.

La configuración está almacenada en el archivo `kind/cluster-multi-ingress.yml`

```shell
$ cat kind/cluster-multi-ingress.yml
---
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
nodes:
  - role: control-plane
  - role: worker
    extraPortMappings:
    - containerPort: 31682
      hostPort: 80
      listenAddress: "127.0.0.1"
      protocol: TCP
```

En la configuración de arriba podemos ver para el role `worker` se define el `extraPortMapping`, lo cual significa
que kind realizará una re dirección de puertos adicional, esta configuración básicamente hace un port forward del
puerto en el host hacia el puerto en un servicio dentro del cluster, los puertos que se re direccionan son:

* TCP `31682` al `80` para acceder a los servicios que expone Kong en modo HTTP

Note también que los puertos que se re direccionan se asocian a la dirección local `127.0.0.1`.

Ahora creamos el cluster versión `1.27.10` con la configuración en el archivo `kind/cluster-multi-ingress.yml`:

```shell
$ kind create cluster --name argo-develop --image kindest/node:v1.27.10 --config=kind/cluster-multi-ingress.yml
Creating cluster "argo-develop" ...
 ✓ Ensuring node image (kindest/node:v1.27.10) 🖼
 ✓ Preparing nodes 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-argo-develop"
You can now use your cluster with:

kubectl cluster-info --context kind-argo-develop

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
```

Listo!! Ya tenemos un cluster con un nodo de control plane y un worker, hagamos un listado de los clusters de kind:

```shell
$ kind get clusters
argo-develop
```

La salida del comando de arriba muestra un cluster llamado `argo-develop`.

Veamos que pasó a nivel contenedores docker:

```shell
$ docker ps
CONTAINER ID   IMAGE                   COMMAND                  CREATED         STATUS         PORTS                                                NAMES
cccb693d433a   kindest/node:v1.27.10   "/usr/local/bin/entr…"   4 minutes ago   Up 3 minutes   127.0.0.1:53476->6443/tcp                            argo-develop-control-plane
e5108810bbf9   kindest/node:v1.27.10   "/usr/local/bin/entr…"   4 minutes ago   Up 3 minutes   127.0.0.1:80->31682/tcp, 127.0.0.1:8001->32581/tcp   argo-develop-worker
```

Arriba se puede ver hay dos contenedores en ejecución asociados a los nodos del cluster.

## Validación del cluster

Además de que el proceso de instalación fue super rápido, kind ya agregó un contexto a la configuración de
`kubectl` local:

```shell
$ kubectl config get-contexts
CURRENT   NAME                       CLUSTER            AUTHINFO              NAMESPACE
*         kind-argo-develop          kind-argo-develop  kind-argo-develop
```

Ahora mostramos la información de dicho cluster:

```shell
$ kubectl cluster-info
Kubernetes control plane is running at https://127.0.0.1:53476
CoreDNS is running at https://127.0.0.1:53476/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Como se pude ver, el cluster está corriendo en `localhost`.

Mostremos la salud del cluster:

```shell
$ kubectl get --raw '/healthz?verbose'
[+]ping ok
[+]log ok
[+]etcd ok
[+]poststarthook/start-kube-apiserver-admission-initializer ok
[+]poststarthook/generic-apiserver-start-informers ok
[+]poststarthook/priority-and-fairness-config-consumer ok
[+]poststarthook/priority-and-fairness-filter ok
[+]poststarthook/storage-object-count-tracker-hook ok
[+]poststarthook/start-apiextensions-informers ok
[+]poststarthook/start-apiextensions-controllers ok
[+]poststarthook/crd-informer-synced ok
[+]poststarthook/start-system-namespaces-controller ok
[+]poststarthook/bootstrap-controller ok
[+]poststarthook/rbac/bootstrap-roles ok
[+]poststarthook/scheduling/bootstrap-system-priority-classes ok
[+]poststarthook/priority-and-fairness-config-producer ok
[+]poststarthook/start-cluster-authentication-info-controller ok
[+]poststarthook/start-kube-apiserver-identity-lease-controller ok
[+]poststarthook/start-deprecated-kube-apiserver-identity-lease-garbage-collector ok
[+]poststarthook/start-kube-apiserver-identity-lease-garbage-collector ok
[+]poststarthook/start-legacy-token-tracking-controller ok
[+]poststarthook/aggregator-reload-proxy-client-cert ok
[+]poststarthook/start-kube-aggregator-informers ok
[+]poststarthook/apiservice-registration-controller ok
[+]poststarthook/apiservice-status-available-controller ok
[+]poststarthook/kube-apiserver-autoregistration ok
[+]autoregister-completion ok
[+]poststarthook/apiservice-openapi-controller ok
[+]poststarthook/apiservice-openapiv3-controller ok
[+]poststarthook/apiservice-discovery-controller ok
healthz check passed
```

Listamos los nodos del cluster:

```shell
$ kubectl get nodes
NAME                         STATUS   ROLES           AGE     VERSION
argo-develop-control-plane   Ready    control-plane   5m42s   v1.27.10
argo-develop-worker          Ready    <none>          5m18s   v1.27.10
```

Como se puede ver tenemos un nodo que es el maestro, es decir, la capa de control, y tenemos otro que es el worker.

Listemos los pods de los servicios que están en ejecución:

```shell
$ kubectl get pods -A
NAMESPACE            NAME                                                READY   STATUS    RESTARTS   AGE
kube-system          coredns-5d78c9869d-bc62w                             1/1     Running   0          5m47s
kube-system          coredns-5d78c9869d-w6629                             1/1     Running   0          5m47s
kube-system          etcd-argo-develop-control-plane                      1/1     Running   0          6m1s
kube-system          kindnet-2gfzn                                        1/1     Running   0          5m41s
kube-system          kindnet-k4zgs                                        1/1     Running   0          5m48s
kube-system          kube-apiserver-argo-develop-control-plane            1/1     Running   0          6m1s
kube-system          kube-controller-manager-argo-develop-control-plane   1/1     Running   0          6m1s
kube-system          kube-proxy-8wcnh                                     1/1     Running   0          5m48s
kube-system          kube-proxy-jjhft                                     1/1     Running   0          5m41s
kube-system          kube-scheduler-argo-develop-control-plane            1/1     Running   0          6m1s
local-path-storage   local-path-provisioner-5b77c697fd-cxw2h              1/1     Running   0          5m47s
```

Esto se ve bien, todos los pods están `Running` :), en su mayoría son los servicios del cluster:

* kube-apiserver
* kube-scheduler
* kube-proxy
* kube-controller-manager
* etcd
* kindnet
* coredns
* local-path-provisioner

Todo indica a que el cluster tiene todo listo para desplegar nuestras aplicaciones.

## Despliegue de Kong

Instalaremos Kong para publicar los diferentes servicios web que desplegaremos en las siguientes secciones.

Usando helm, agregamos el repositorio de kong y actualizamos los repos locales:

```shell
helm repo add kong https://charts.konghq.com
helm repo update
```

Creamos el namespace `kong`:

```shell
kubectl create namespace kong
```

Ejecutamos la instalación con los parámetros personalizados para habilitar el servicio de admin y postgresql:

```shell
helm install api-gateway kong/kong -n kong \
  --set manager.enabled=false \
  --set proxy.enabled=true \
  --set proxy.type=NodePort \
  --set proxy.http.enabled=true \
  --set proxy.http.nodePort=31682
```

Listemos los recursos en el namespace de kong:

```shell
$ kubectl -n kong get pods,services
NAME                                    READY   STATUS    RESTARTS   AGE
pod/api-gateway-kong-6c849d74c5-57v49   0/2     Running   0          6s

NAME                                         TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)                     AGE
service/api-gateway-kong-proxy               NodePort   10.96.242.235  <none>       80:31682/TCP,443:30803/TCP  6s
service/api-gateway-kong-validation-webhook  ClusterIP  10.96.59.76    <none>       443/TCP                     6s
```

Muy bien, en la salida de arriba vemos el pod del `api-gateway-kong` en estado running, y en los servicios,
tenemos el del proxy de tipo `NodePort`, mapeando el puerto `80` al `31682`.

Hagamos una petición a kong al puerto TCP/80 donde se exponen los servicios:

```shell
$ curl http://localhost
{
  "message":"no Route matched with those values",
  "request_id":"9ef3133f3f3eab1d4f142f567e7c4156"
}
```

Teniendo una respuesta como la de arriba terminamos esta sección.

## Despliegue de ArgoCD

Creamos un namespace dedicado para ArgoCD:

```shell
$ kubectl create namespace argocd
namespace/argocd created
```

Ahora instalamos ArgoCD stable:

```shell
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
customresourcedefinition.apiextensions.k8s.io/applications.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/applicationsets.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/appprojects.argoproj.io created
serviceaccount/argocd-application-controller created
serviceaccount/argocd-applicationset-controller created
serviceaccount/argocd-dex-server created
serviceaccount/argocd-notifications-controller created
serviceaccount/argocd-redis created
serviceaccount/argocd-repo-server created
serviceaccount/argocd-server created
role.rbac.authorization.k8s.io/argocd-application-controller created
role.rbac.authorization.k8s.io/argocd-applicationset-controller created
role.rbac.authorization.k8s.io/argocd-dex-server created
role.rbac.authorization.k8s.io/argocd-notifications-controller created
role.rbac.authorization.k8s.io/argocd-server created
clusterrole.rbac.authorization.k8s.io/argocd-application-controller created
clusterrole.rbac.authorization.k8s.io/argocd-applicationset-controller created
clusterrole.rbac.authorization.k8s.io/argocd-server created
rolebinding.rbac.authorization.k8s.io/argocd-application-controller created
rolebinding.rbac.authorization.k8s.io/argocd-applicationset-controller created
rolebinding.rbac.authorization.k8s.io/argocd-dex-server created
rolebinding.rbac.authorization.k8s.io/argocd-notifications-controller created
rolebinding.rbac.authorization.k8s.io/argocd-server created
clusterrolebinding.rbac.authorization.k8s.io/argocd-application-controller created
clusterrolebinding.rbac.authorization.k8s.io/argocd-applicationset-controller created
clusterrolebinding.rbac.authorization.k8s.io/argocd-server created
configmap/argocd-cm created
configmap/argocd-cmd-params-cm created
configmap/argocd-gpg-keys-cm created
configmap/argocd-notifications-cm created
configmap/argocd-rbac-cm created
configmap/argocd-ssh-known-hosts-cm created
configmap/argocd-tls-certs-cm created
secret/argocd-notifications-secret created
secret/argocd-secret created
service/argocd-applicationset-controller created
service/argocd-dex-server created
service/argocd-metrics created
service/argocd-notifications-controller-metrics created
service/argocd-redis created
service/argocd-repo-server created
service/argocd-server created
service/argocd-server-metrics created
deployment.apps/argocd-applicationset-controller created
deployment.apps/argocd-dex-server created
deployment.apps/argocd-notifications-controller created
deployment.apps/argocd-redis created
deployment.apps/argocd-repo-server created
deployment.apps/argocd-server created
statefulset.apps/argocd-application-controller created
networkpolicy.networking.k8s.io/argocd-application-controller-network-policy created
networkpolicy.networking.k8s.io/argocd-applicationset-controller-network-policy created
networkpolicy.networking.k8s.io/argocd-dex-server-network-policy created
networkpolicy.networking.k8s.io/argocd-notifications-controller-network-policy created
networkpolicy.networking.k8s.io/argocd-redis-network-policy created
networkpolicy.networking.k8s.io/argocd-repo-server-network-policy created
networkpolicy.networking.k8s.io/argocd-server-network-policy created
```

Verificamos los pods de argocd:

```shell
$ kubectl get pods -n argocd
NAME                                               READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                    1/1     Running   0          43s
argocd-applicationset-controller-79c95f5d7-mpkd7   1/1     Running   0          44s
argocd-dex-server-f5d97b5b-p54ph                   1/1     Running   0          44s
argocd-notifications-controller-7f8d9dd7f-wsbsr    1/1     Running   0          44s
argocd-redis-69f8795dbd-t9bk2                      1/1     Running   0          43s
argocd-repo-server-9cf5d5585-97hwk                 1/1     Running   0          43s
argocd-server-7574cff9df-b8rkh                     0/1     Running   0          43s
```

Verificamos los servicios de argocd:

```shell
$ kubectl get services -n argocd
NAME                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
argocd-applicationset-controller          ClusterIP   10.96.151.142   <none>        7000/TCP,8080/TCP            69s
argocd-dex-server                         ClusterIP   10.96.233.6     <none>        5556/TCP,5557/TCP,5558/TCP   69s
argocd-metrics                            ClusterIP   10.96.54.13     <none>        8082/TCP                     69s
argocd-notifications-controller-metrics   ClusterIP   10.96.209.232   <none>        9001/TCP                     69s
argocd-redis                              ClusterIP   10.96.12.103    <none>        6379/TCP                     69s
argocd-repo-server                        ClusterIP   10.96.29.136    <none>        8081/TCP,8084/TCP            69s
argocd-server                             ClusterIP   10.96.47.189    <none>        80/TCP,443/TCP               69s
argocd-server-metrics                     ClusterIP   10.96.125.85    <none>        8083/TCP                     69s
```

Para conectarnos al portal web de ArgoCD vamos a realizar una re dirección de puertos o Port forwarding:

```shell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Loa anterior mapea el puerto `8080` del localhost al puerto 443 del servicio `argocd-server`, para verificarlo
abre en tu navegador el siguiente URL: [https://localhost:8080](https://localhost:8080/).

## Configuración de repositorio Git

Entramos al UI, vamos al menú `Settings`, `Repositories` y hacemos clic en `CONNECT REPO`, y definimos los
siguientes parámetros:

* Choose the connection method: VIA HTTPS
* Type: git
* Project: default
* Repository URL: https://github.com/jorgearma1982/gitops-k8s-argocd-infra-configs.git

Es todo, hacemos clic en `CONNECT` para terminar, al final debemos ver el estado de conexión en `Success`.

### Despliegue de aplicaciones

Para desplegar una aplicación vamos al menú `Applications` y hacemos clic en `NEW APP`:

En la sección `General` usamos:

* Application Name: whoami
* Project: Default
* Sync Policy: Manual

En la sección `Source` usamos:

* Repository URL: https://github.com/jorgearma1982/gitops-k8s-argocd-infra-configs.git
* Revision: HEAD
* Path: kubernetes/whoami

En la sección `Destination` usamos:

* Cluster URL: https://kubernetes.default.svc
* Namespace: default

Para aplicar los cambios hacemos clic en `CREATE`.

Si las configuraciones son correctos se agrega la aplicación y por default aparece en estado `OutOfSync`, ahora
seleccionamos la aplicación y hacemos clic en `DETAILS`, después en `EDIT` y modificamos `LABELS` con los
siguientes datos:

* project = argocd
* environment = develop
* team = platform
* owner = jorge-medina

Terminamos de editar las etiquetas y hacemos clic en `Save`.

Lo siguiente es hacer clic en el botón `SYNC` para sincronizar el cluster con el repositorio Git.

## Validación de aplicación

Ahora validamos listando todos los recursos del namespace `default`:

```shell
$ kubectl -n default get all
NAME                          READY   STATUS    RESTARTS   AGE
pod/whoami-6977d564f9-dq5pr   1/1     Running   0          2m35s

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP    4m51s
service/whoami       ClusterIP   10.96.27.189   <none>        8080/TCP   2m24s

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/whoami   1/1     1            1           2m35s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/whoami-6977d564f9   1         1         1       2m35s
```

Como se puede ver se tiene los recursos `deployment`, el `replicaset`, los `pods` y el `service`.

Ahora validamos que responda el servicio whoami a través de kong:

```shell
$ curl http://localhost/whoami
Hostname: whoami-6977d564f9-dq5pr
IP: 127.0.0.1
IP: ::1
IP: 10.244.1.3
IP: fe80::dc3f:a9ff:fe84:9283
RemoteAddr: 10.244.1.2:58086
GET / HTTP/1.1
Host: localhost
User-Agent: curl/7.64.1
Accept: */*
Connection: keep-alive
X-Forwarded-For: 10.244.1.1
X-Forwarded-Host: localhost
X-Forwarded-Path: /whoami
X-Forwarded-Port: 80
X-Forwarded-Prefix: /whoami
X-Forwarded-Proto: http
X-Real-Ip: 10.244.1.1
```

Listo!, ya tenemos una respuesta de `whoami`.

## Pruebas de carga a la aplicación web

Usaremos `k6` para realizar pruebas de carga en la aplicación que exponemos a través de kong:

Ahora ejecutamos el script con las pruebas:

```shell
$ k6 run k6/script.js
```

## Limpieza

Para destruir el cluster ejecutamos:

```shell
$ kind delete cluster --name argo-develop
Deleting cluster "argo-develop" ...
```

También podemos limpiar colima:

```shell
$ colima delete
are you sure you want to delete colima and all settings? [y/N] y
INFO[0001] deleting colima
INFO[0001] deleting ...                                  context=docker
INFO[0001] done
```

Y listo todo se ha terminado.

## Problemas conocidos

Si usas una mac m1 es probable que tengas errores al descargar las imágenes de los contenedores, si es un error
relacionado a resolución de nombres DNS, puedes probar agregando la configuración de `lima` para que no use
el dns del host y en su lugar use el de google, por ejemplo:

Creamos configuración para dns de lima:

```shell
$ vim ~/.lima/_config/override.yaml
```

Con el siguiente contenido:

```shell
useHostResolver: false
dns:
  - 8.8.8.8
```

Se recomienda que hagas un reset de colima, haciendo delete, y nuevamente start.

También puedes iniciar colima con la opción `--dns`, por ejemplo:

```shell
$ colima start --dns 8.8.8.8
```

## Comandos útiles

Listado versiones:

* kubectl version

Listado contextos:

* kubectl config get-contexts

Detalles de cluster:

* kubectl cluster-info

Gestión de nodos:

* kubectl get nodes
* kubectl describe node NODENAME

Gestión de pods:

* kubectl get pods
* kubectl describe pod PODNAME
* kubectl logs PODNAME
* kubectl delete pod PODNAME

Gestión de services:

* kubectl get services
* kubectl describe service SVCNAME
* kubectl delete service SVCNAME

Gestión de namespaces:

* kubectl get namespaces
* kubectl describe namespace NSNAME
* kubectl delete namespace NSNAME

Gestión de recursos en modo declarativo:

* kubectl apply -f YAMLFILE
* kubectl delete -f YAMLFILE

Gestión de deployments:

* kubectl get deployment
* kubectl describe deployment PODNAME
* kubectl delete deployment PODNAME

Gestión charts:

* helm ls
* helm install CHARTNAME
* helm uninstall CHARTNAME

## Referencias

La siguiente es una lista de referencias externas que pueden serle de utilidad:

* [Colima - container runtimes on macOS (and Linux) with minimal setup](https://github.com/abiosoft/colima)
* [Kubernetes - Orquestación de contenedores para producción](https://kubernetes.io/es/)
* [kind - home](https://kind.sigs.k8s.io/)
* [kind - quick start](https://kind.sigs.k8s.io/docs/user/quick-start/)
* ArgoCD
* [Kong Ingress Controller](https://docs.konghq.com/kubernetes-ingress-controller/latest/)
* [Kong - Getting started guide](https://docs.konghq.com/kubernetes-ingress-controller/latest/guides/getting-started/)
