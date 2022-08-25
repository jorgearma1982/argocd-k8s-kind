# ArgoCD en Kubernetes con kind

## Introducción

En este guía mostramos como instalar un cluster de `Kubernetes` en la laptop usando la implementación
de `kind`, la cual corre cada componente en un contenedor en lugar de usar máquinas virtuales, originalmente
fue diseñado para probar kubernetes en sí, pero también puede ser usado para desarrollo local ó CI.

Este proyecto puede servir para comprender los conceptos, la arquitectura y adentrarnos más en lo que
son los contenedores, los pods y su relación con los microservicios.

Instalaremos `ArgoCD` 

Instalaremos `Kong` como Ingress Controller y una aplicación web simple para validar la funcionalidad de Kong
como API Gateway.

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
INFO[0000] preparing network ...                         context=vm
INFO[0000] creating and starting ...                     context=vm
INFO[0023] provisioning ...                              context=docker
INFO[0023] starting ...                                  context=docker
INFO[0028] done
```

**NOTA:** Por default colima levanta una máquina virtual con `2` vCPUs y `2` GB de RAM, si se desea modificar
esto para asignar más CPU o RAM, puedes agregar los parámetros `--cpu 4` y `--memory 4`.

Ahora instalamos los paquetes para kubernetes con `kind`, también instalamos el cliente `kubectl` y
`k6` la herramienta de pruebas de carga de aplicaciones web:

```shell
$ brew install kind kubectl k6
```

Validamos la instalación de las herramientas, iniciamos con kind:

```shell
$ kind --version
kind version 0.14.0
```

Ahora veamos la versión de `kubectl`:

```shell
$ kubectl version --client=true
Client Version: version.Info{Major:"1", Minor:"24", GitVersion:"v1.24.3"}
Kustomize Version: v4.5.4
```

Y finalmente la versión de `k6`:

```shell
$ k6 version
k6 v0.39.0 ((devel)) 
```

## Instalación de cluster

Definimos la configuración del cluster con dos nodos, uno con rol de `control-plane` y otro de `worker`.

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
    - containerPort: 32581
      hostPort: 8001
      listenAddress: "127.0.0.1"
      protocol: TCP
```

En la configuración de arriba podemos ver para el role `worker` se define el `extraPortMapping`, lo cual significa
que kind realizará una re dirección de puertos adicional, esta configuración básicamente hace un port forward del
puerto en el host hacia el puerto en un servicio dentro del cluster, los puertos que se redireccionan son:

* TCP `31682` al `80` para acceder a los servicios que expone Kong en modo HTTP
* TCP `32581` al `8001` para acceder a la API de administración de Kong en modo HTTP.

Note también que los puertos que se re direccionan se asocian a la dirección local `127.0.0.1`.

Ahora creamos el cluster versión `1.23.4` con la configuración en el archivo `kind/cluster-multi-ingress.yml`:

```shell
$ kind create cluster --name argocluster --image kindest/node:v1.23.4 --config=kind/cluster-multi-ingress.yml
Creating cluster "argocluster" ...
 ✓ Ensuring node image (kindest/node:v1.23.4) 🖼
 ✓ Preparing nodes 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-argocluster"
You can now use your cluster with:

kubectl cluster-info --context kind-argocluster

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/😊
```

Listo!! Ya tenemos un cluster con un nodo de control plane y un worker, hagamos un listado de los clusters de kind:

```shell
$ kind get clusters
argocluster
```

La salida del comando de arriba muestra un cluster llamado `argocluster`.

Veamos que pasó a nivel contenedores docker:

```shell
$ docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED          STATUS          PORTS                                                NAMES
333300549e53   kindest/node:v1.23.4   "/usr/local/bin/entr…"   13 minutes ago   Up 13 minutes   127.0.0.1:80->31682/tcp, 127.0.0.1:8001->32581/tcp   argocluster-worker
6d8a056a6db9   kindest/node:v1.23.4   "/usr/local/bin/entr…"   13 minutes ago   Up 13 minutes   127.0.0.1:60836->6443/tcp                            argocluster-control-plane
```

Arriba se puede ver hay dos contenedores en ejecución asociados a los nodos del cluster.

## Validación del cluster

Además de que el proceso de instalación fue super rápido, kind ya agregó un contexto a la configuración de
`kubectl` local:

```shell
$ kubectl config get-contexts
CURRENT   NAME                       CLUSTER            AUTHINFO              NAMESPACE
*         kind-argocluster           kind-argocluster   kind-argocluster
```

Ahora mostramos la información de dicho cluster:

```shell
$ kubectl cluster-info
Kubernetes control plane is running at https://127.0.0.1:53551
CoreDNS is running at https://127.0.0.1:53551/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

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
[+]poststarthook/start-apiextensions-informers ok
[+]poststarthook/start-apiextensions-controllers ok
[+]poststarthook/crd-informer-synced ok
[+]poststarthook/bootstrap-controller ok
[+]poststarthook/rbac/bootstrap-roles ok
[+]poststarthook/scheduling/bootstrap-system-priority-classes ok
[+]poststarthook/priority-and-fairness-config-producer ok
[+]poststarthook/start-cluster-authentication-info-controller ok
[+]poststarthook/aggregator-reload-proxy-client-cert ok
[+]poststarthook/start-kube-aggregator-informers ok
[+]poststarthook/apiservice-registration-controller ok
[+]poststarthook/apiservice-status-available-controller ok
[+]poststarthook/kube-apiserver-autoregistration ok
[+]autoregister-completion ok
[+]poststarthook/apiservice-openapi-controller ok
healthz check passed
```

Listamos los nodos del cluster:

```shell
$ kubectl get nodes
NAME                        STATUS   ROLES                  AGE   VERSION
argocluster-control-plane   Ready    control-plane,master   55m   v1.23.4
argocluster-worker          Ready    <none>                 54m   v1.23.4
```

Como se puede ver tenemos un nodo que es el maestro, es decir, la capa de control, y tenemos otro que es el worker.

Listemos los pods de los servicios que están en ejecución:

```shell
$ kubectl get pods -A
NAMESPACE            NAME                                                READY   STATUS    RESTARTS   AGE
kube-system          coredns-64897985d-4ddkv                             1/1     Running   0          56m
kube-system          coredns-64897985d-t2jrp                             1/1     Running   0          56m
kube-system          etcd-argocluster-control-plane                      1/1     Running   0          56m
kube-system          kindnet-b2jfz                                       1/1     Running   0          55m
kube-system          kindnet-n6tsn                                       1/1     Running   0          56m
kube-system          kube-apiserver-argocluster-control-plane            1/1     Running   0          56m
kube-system          kube-controller-manager-argocluster-control-plane   1/1     Running   0          56m
kube-system          kube-proxy-fdcb6                                    1/1     Running   0          56m
kube-system          kube-proxy-vt2pm                                    1/1     Running   0          55m
kube-system          kube-scheduler-argocluster-control-plane            1/1     Running   0          56m
local-path-storage   local-path-provisioner-5ddd94ff66-mc86h             1/1     Running   0          56m
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
clusterrole.rbac.authorization.k8s.io/argocd-server created
rolebinding.rbac.authorization.k8s.io/argocd-application-controller created
rolebinding.rbac.authorization.k8s.io/argocd-applicationset-controller created
rolebinding.rbac.authorization.k8s.io/argocd-dex-server created
rolebinding.rbac.authorization.k8s.io/argocd-notifications-controller created
rolebinding.rbac.authorization.k8s.io/argocd-redis created
rolebinding.rbac.authorization.k8s.io/argocd-server created
clusterrolebinding.rbac.authorization.k8s.io/argocd-application-controller created
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
networkpolicy.networking.k8s.io/argocd-dex-server-network-policy created
networkpolicy.networking.k8s.io/argocd-redis-network-policy created
networkpolicy.networking.k8s.io/argocd-repo-server-network-policy created
networkpolicy.networking.k8s.io/argocd-server-network-policy created
```

Verificamos los pods de argocd:

```shell
$ kubectl get pods -n argocd
NAME                                               READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                    1/1     Running   0          89s
argocd-applicationset-controller-7f466f7cc-z4v2c   1/1     Running   0          90s
argocd-dex-server-54cd4596c4-57rwp                 1/1     Running   0          90s
argocd-notifications-controller-8445d56d96-7z2v6   1/1     Running   0          89s
argocd-redis-65596bf87-nxqgf                       1/1     Running   0          89s
argocd-repo-server-5ccf4bd568-cgp9n                0/1     Running   0          89s
argocd-server-7dff66c8f8-bmt8l                     1/1     Running   0          89s
```

Verificamos los servicios de argocd:

```shell
$ kubectl get services -n argocd
NAME                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
argocd-applicationset-controller          ClusterIP   10.96.168.2     <none>        7000/TCP,8080/TCP            109s
argocd-dex-server                         ClusterIP   10.96.121.150   <none>        5556/TCP,5557/TCP,5558/TCP   109s
argocd-metrics                            ClusterIP   10.96.19.211    <none>        8082/TCP                     109s
argocd-notifications-controller-metrics   ClusterIP   10.96.16.194    <none>        9001/TCP                     109s
argocd-redis                              ClusterIP   10.96.245.35    <none>        6379/TCP                     109s
argocd-repo-server                        ClusterIP   10.96.214.212   <none>        8081/TCP,8084/TCP            109s
argocd-server                             ClusterIP   10.96.212.131   <none>        80/TCP,443/TCP               109s
argocd-server-metrics                     ClusterIP   10.96.176.164   <none>        8083/TCP                     109s
```

## Despliegue de Kong

Instalaremos Kong con base de datos postgres para almacenar todas las configuraciones.

Usando helm, agregamos el repositorio de kong:

```shell
$ helm repo add kong https://charts.konghq.com
```

Ahora actualizamos los repositorios:

```shell
$ helm repo update
```

Creamos un namespace para kong:

```shell
$ kubectl create namespace kong
namespace/kong created
```

Listamos los namespaces:

```shell
$ kubectl get namespace kong
NAME   STATUS   AGE
kong   Active   1s
```

Ejecutamos la instalación con los parámetros personalizados para habilitar el servicio de admin y postgresql:

```shell
$ helm install api-gateway -n kong kong/kong \
  --set ingressController.installCRDs=false \
  --set admin.enabled=true \
  --set admin.type=ClusterIP \
  --set admin.http.enabled=true \
  --set admin.tls.enabled=false \
  --set env.database=postgres \
  --set postgresql.enabled=true
  NAME: api-gateway
LAST DEPLOYED: Fri Aug  5 09:51:02 2022
NAMESPACE: kong
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
To connect to Kong, please execute the following commands:

HOST=$(kubectl get svc --namespace kong api-gateway-kong-proxy -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
PORT=$(kubectl get svc --namespace kong api-gateway-kong-proxy -o jsonpath='{.spec.ports[0].port}')
export PROXY_IP=${HOST}:${PORT}
curl $PROXY_IP

Once installed, please follow along the getting started guide to start using
Kong: https://docs.konghq.com/kubernetes-ingress-controller/latest/guides/getting-started/
```

Esta instalación de Kong crea varios recursos de tipo `CRD` ó `Custom Resource Definition` que se usan
para configurar los recursos de kong de forma declarativa. Además se crean cuentas de servicio y los roles
de acceso, así como un secreto, los servicios de red, el deployment y un ingress class.

Listemos los recursos en el namespace de kong:

```shell
$ kubectl -n kong get all
NAME                                         READY   STATUS      RESTARTS   AGE
pod/api-gateway-kong-5f8d97b9c5-dp4rw        2/2     Running     0          7m
pod/api-gateway-kong-init-migrations-gmc7s   0/1     Completed   0          7m
pod/api-gateway-postgresql-0                 1/1     Running     0          7m

NAME                                TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/api-gateway-kong-admin      ClusterIP      10.96.100.125   <none>        8001/TCP                     7m
service/api-gateway-kong-proxy      LoadBalancer   10.96.39.118    <pending>     80:30448/TCP,443:32649/TCP   7m
service/api-gateway-postgresql      ClusterIP      10.96.10.142    <none>        5432/TCP                     7m
service/api-gateway-postgresql-hl   ClusterIP      None            <none>        5432/TCP                     7m

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/api-gateway-kong   1/1     1            1           7m

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/api-gateway-kong-5f8d97b9c5   1         1         1       7m

NAME                                      READY   AGE
statefulset.apps/api-gateway-postgresql   1/1     7m

NAME                                         COMPLETIONS   DURATION   AGE
job.batch/api-gateway-kong-init-migrations   1/1           40s        7m
```

En el listado vemos que hay un deployment llamado `api-gateway-kong`, el cual los siguientes pods:

* api-gateway-kong
* api-gateway-kong-init-migrations
* api-gateway-postgresql

La base de datos se levanta y se configura con credenciales default en el pod `api-gateway-postgresql`. El pod
`api-gateway-initmigrations` solo se ejecuta una vez para inicializar y configurar la base de datos postgres.
Finalmente el pod `api-gateway-kong` es el servidor principal de kong.

En el listado de servicios, el servicio `api-gateway-kong-admin` es de tipo `ClusterIP` en el puerto `8001`. También
se puede ver el servicio `api-gateway-kong-proxy` de tipo `LoadBalancer` y mapea el puerto `30448` al `80`,
y el `30428` al `443` en TCP. Necesitamos cambiar esos puertos para que hagan coincidencia con los que definimos
en el port mapping al crear el cluster.

Actualizamos la configuración del servicio kong admin:

```shell
$ kubectl -n kong patch service api-gateway-kong-admin --patch-file kong/patch-service-admin-nodeport.yml
service/api-gateway-kong-admin patched
```

También actualizamos el servicio kong proxy:

```shell
$ kubectl -n kong patch service api-gateway-kong-proxy --patch-file kong/patch-service-proxy-nodeport.yml
service/api-gateway-kong-proxy patched
```

Ahora listamos para ver los cambios:

```shell
$ kubectl -n kong get services | grep api-gateway-kong
api-gateway-kong-admin      NodePort    10.96.157.9     <none>        8001:32581/TCP               12m
api-gateway-kong-proxy      NodePort    10.96.209.152   <none>        80:31682/TCP,443:32527/TCP   12m
```

Como se puede ver ya se usan los puertos que definimos al inicio.

Hagamos una petición a kong al puerto TCP/80 donde se exponen los servicios:

```shell
$ curl http://localhost/
{"message":"no Route matched with those values"}
```

También podemos hacer una petición a la API de administración de Kong:

```shell
$ curl http://localhost:8001/
{"configuration":{"go_pluginserver_exe":"/usr/local/bin/go-pluginserver","mem_cache_size":"128m","db_cache_warmup_entities":["services"],"headers":["server_tokens","latency_tokens"],"worker_consistency":"strict","nginx_events_directives":[{"name":"multi_accept","value":"on"},{"name":"worker_connections","value":"auto"}],"nginx_http_directives":[{"name":"client_body_buffer_size","value":"8k"},{"name":"client_max_body_size","value":"0"}..."tagline":"Welcome to kong","timers":{"running":0,"pending":10},"pids":{"workers":[1114,1115],"master":1}}
```

Listo!!! Ya tenemos Kong instalado y listo para usarse.

## Despliegue aplicación

Realizamos el despliegue de una aplicación:

```shell
$ kubectl apply -f whoami/1_deployment.yml
```

Creamos el service de una aplicación:

```shell
$ kubectl apply -f whoami/2_service.yml
```

Creamos el ingress de una aplicación:

```shell
$ kubectl apply -f whoami/3_ingress.yml
```

Esperamos unos segundos a que levanten los servicios y continuamos con la validaciones.

## Validación aplicación

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
$ kind delete cluster --name argocluster
Deleting cluster "argocluster" ...
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
