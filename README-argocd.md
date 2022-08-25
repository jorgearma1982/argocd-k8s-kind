---
layout: page
title: Despliegues con Argo CD
parent: Continuous Delivery
nav_order: 11
---

## Despliegues con Argo CD

En esta guía describimos los diferentes procesos y herramientas que usamos para realizar el despliegue de las
aplicaciones de la plataforma KronOps a el cluster de contenedores Kubernetes usando la herramienta de entrega
continua `Argo CD`.

### Introducción

Las definiciones de las aplicaciones, sus configuraciones, y los ambientes deberían ser declarativos y con control
de versiones. Los despliegues de las aplicaciones y la gestión de ciclo de vida debería ser automatizado, auditable,
y fácil de entender.

Argo CD es una herramienta para Despliegues Continuos (CD) nativa de Kubernetes. A diferencia de otras herramientas
de CD que solo hacen despliegues en un esquema basado en `push`, Argo CD puede hacer `pull` de código de actualizado
desde un repositorio Git y desplegarlo directamente a Kubernetes. Permite a los desarrolladores manejar
las configuraciones de infraestructura y las actualizaciones de las aplicaciones en un solo sistema.

Argo CD ofrece las siguientes funcionalidades y capacidades clave:

 * Despliegues manuales y automáticos de aplicaciones a un cluster Kubernetes.
 * Sincronización automática del estado de una aplicación con la versión actual de una configuración declarativa.
 * Interfaz de usuario web y de línea de comandos.
 * Facilidad para visualizar problemas en los despliegues, detectar y remediar variaciones en las configuraciones.
 * El control de acceso basado en roles permite la administración de múltiples clusters.
 * Single Sign-on (SSO) con proveedores para GitLab, Github, OAuth2, OIDC, LDAP y SAML 2.0.
 * Soporte para webhooks que disparan acciones en GitLab, GitHub y BitBucket.

Usaremos el patrón `GitOps` para definir el estado deseado de las aplicaciones en el cluster, esto quiere decir que
el repositorio Git de las configuraciones será la única fuente de la verdad. Los manifiestos de Kubernetes pueden
ser especificados de las siguientes formas:

 * Aplicaciones `kustomize`.
 * helm charts.
 * Aplicaciones `ksonnet`.
 * Archivos `jsonnet`.
 * Un directorio con manifiestos `YAML`.
 * Cualquier herramienta de gestión de configuraciones personalizada para usarse como plugin.

#### Arquitectura

En el siguiente diagrama se muestra la arquitectura de Argo CD en relación a los procesos de CD/CD:

![Argo CD Architecture](/assets/images/argocd_architecture.png)

#### Objetivos

Describiremos los requisitos necesarios para instalar y configurar Argo CD para automatizar el despliegue de
las aplicaciones que se definen en un repositorio Git. En este caso empezaremos con el ambiente de pruebas `Sandbox`.

Los objetivos clave son:

 * Definir requisitos
 * Instalar Argo CD en Kubernetes
 * Configurar Argo CD
 * Configurar repositorio git de configuraciones
 * Dar de alta aplicaciones
 * Realizar respaldo

### Requisitos

Para poder instalar y usar Argo CD vamos a requerir lo siguiente:

 * Repositorio Git con configuraciones de aplicaciones
 * Cuenta de acceso al repositorio git de configuraciones con llave ssh
 * Cluster Kubernetes en Operación
 * Cuenta de acceso al cluster de kubernetes como administrador

### Instalando Argo CD

Comenzaremos por crear un espacio de nombres para la aplicación Argo CD:

```shell
$ kubectl create namespace argocd
namespace/argocd created
```

Instalamos Argo CD desde los manifiestos oficiales:

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

### Verificamos la instalación

El manifiesto de instalación oficial instala diferentes recursos, verificamos despliegue haciendo un listado
de los despliegues que se crearon:

```shell
$ kubectl -n argocd get deployments
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
argocd-applicationset-controller   1/1     1            1           29h
argocd-dex-server                  1/1     1            1           29h
argocd-notifications-controller    1/1     1            1           29h
argocd-redis                       1/1     1            1           29h
argocd-repo-server                 1/1     1            1           29h
argocd-server                      1/1     1            1           29h
```

Ahora verificamos que los pods estén en un estado de ejecución:

```shell
$ kubectl -n argocd get pods
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          4h32m
argocd-applicationset-controller-66689cbf4b-bchwx   1/1     Running   0          4h32m
argocd-dex-server-66fc6c99cc-jjwj2                  1/1     Running   0          18h
argocd-notifications-controller-8f8f46bd6-vg6hj     1/1     Running   0          19h
argocd-redis-d486999b7-g575j                        1/1     Running   0          4h32m
argocd-repo-server-7db4cc4b45-jzbzr                 1/1     Running   0          18h
argocd-server-79d86bbb47-z9rcw                      1/1     Running   0          19h
```

Y los servicios correspondientes:

```shell
$ kubectl -n argocd get services
NAME                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
argocd-applicationset-controller          ClusterIP   10.228.9.131    <none>        7000/TCP                     29h
argocd-dex-server                         ClusterIP   10.228.7.165    <none>        5556/TCP,5557/TCP,5558/TCP   29h
argocd-metrics                            ClusterIP   10.228.3.145    <none>        8082/TCP                     29h
argocd-notifications-controller-metrics   ClusterIP   10.228.12.148   <none>        9001/TCP                     29h
argocd-redis                              ClusterIP   10.228.9.137    <none>        6379/TCP                     29h
argocd-repo-server                        ClusterIP   10.228.11.31    <none>        8081/TCP,8084/TCP            29h
argocd-server                             ClusterIP   10.228.9.45     <none>        80/TCP,443/TCP               29h
argocd-server-metrics                     ClusterIP   10.228.3.207    <none>        8083/TCP                     29h
```

Como podemos tenemos varios servidores que conforman un despliegue estándar de Argo CD.

### Configurando el Ingress para Argo CD

Crearemos un Ingress para exponer el servicio al exterior:

```shell
$ vim 3_ingress.yaml
```

Agregamos el siguiente contenido:

```shell
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server
  namespace: argocd
  annotations:
    kubernetes.io/tls-acme: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    konghq.com/protocols: "http,https"
    konghq.com/https-redirect-status-code: "301"
spec:
  ingressClassName: kong
  rules:
    - host: cd.apis.example.io
      http:
        paths:
          - path: /
            pathType: ImplementationSpecific
            backend:
              service:
                name: argocd-server
                port:
                  name: https
  tls:
    - secretName: kronops-apis-cd-acme-certificate
      hosts:
        - cd.apis.example.io
```

**NOTA:** Los annotations de `konghq.com` de `protocol` y `https-redirect-status-code` permiten la comunicación
icon el backend de Argo CD por https.

Aplicamos el ingress:

```shell
$ k apply -f 3_ingress.yaml
```

Verificamos el ingress:

```shell
$ k -n argocd get ingress
NAME                   CLASS   HOSTS                 ADDRESS         PORTS     AGE
kronops-apis-argocd   kong    cd.apis.example.io   35.192.43.151   80, 443   66m
```

Ahora parchamos el service `argocd-server` para agregar un annotation e indicarle a kong que este es un servicio
de tipo `https`:

```shell
$ kubectl -n argocd patch ingress argocd \
  --type='json' -p='[{"op": "add", "path": "/metadata/annotations/konghq.com~1protocol", "value":"https"}]'
```

Con esto el servicio estará disponible en el URL: `https://cd.apis.example.io`.

### Argo CD CLI

Obtenemos la contraseña inicial del administrador de Argo CD:

```shell
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Instalamos el CLI:

```shell
$ brew install argocd
```

Iniciar sesión:

```shell
$ argocd login cd.apis.example.io --grpc-web
Username: admin
Password:
'admin:login' logged in successfully
Context 'cd.apis.example.io' updated
```

Actualizamos contraseña:

```shell
$ argocd account update-password --grpc-web
*** Enter password of currently logged in user (admin): ********
*** Enter new password for user admin: ********
*** Confirm new password for user admin: ********
Password updated
Context 'cd.apis.example.io' updated
```

Ahora puede eliminar el secret `argocd-initial-admin-secret`:

```shell
$ k -n argocd delete secret argocd-initial-admin-secret
secret "argocd-initial-admin-secret" deleted
```

Listamos los clusters que estén configurados:

```shell
$ argocd cluster list --grpc-web
SERVER                          NAME        VERSION  STATUS   MESSAGE   PROJECT
https://kubernetes.default.svc  in-cluster           Unknown  Cluster has no application and not being monitored.
```

Agregamos una aplicación:

```shell
$ argocd app create guestbook --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --grpc-web
application 'guestbook' created
```

Listamos la aplicación:

```shell
$ argocd app get guestbook
Name:               guestbook
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://cd.apis.example.io/applications/guestbook
Repo:               https://github.com/argoproj/argocd-example-apps.git
Target:
Path:               guestbook
SyncWindow:         Sync Allowed
Sync Policy:        <none>
Sync Status:        OutOfSync from  (53e28ff)
Health Status:      Missing

GROUP  KIND        NAMESPACE  NAME          STATUS     HEALTH   HOOK  MESSAGE
       Service     default    guestbook-ui  OutOfSync  Missing
apps   Deployment  default    guestbook-ui  OutOfSync  Missing
```

Sincronizamos la aplicación:

```shell
$ argocd app sync guestbook
TIMESTAMP                  GROUP        KIND   NAMESPACE                  NAME    STATUS    HEALTH        HOOK  MESSAGE
2022-06-04T18:38:35-05:00            Service     default          guestbook-ui  OutOfSync  Missing
2022-06-04T18:38:35-05:00   apps  Deployment     default          guestbook-ui  OutOfSync  Missing
2022-06-04T18:38:35-05:00            Service     default          guestbook-ui  OutOfSync  Missing              service/guestbook-ui created
2022-06-04T18:38:35-05:00   apps  Deployment     default          guestbook-ui  OutOfSync  Missing              deployment.apps/guestbook-ui created
2022-06-04T18:38:35-05:00            Service     default          guestbook-ui    Synced  Healthy                  service/guestbook-ui created
2022-06-04T18:38:35-05:00   apps  Deployment     default          guestbook-ui    Synced  Progressing              deployment.apps/guestbook-ui created

Name:               guestbook
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://cd.apis.example.io/applications/guestbook
Repo:               https://github.com/argoproj/argocd-example-apps.git
Target:
Path:               guestbook
SyncWindow:         Sync Allowed
Sync Policy:        <none>
Sync Status:        Synced to  (53e28ff)
Health Status:      Healthy

Operation:          Sync
Sync Revision:      53e28ff20cc530b9ada2173fbbd64d48338583ba
Phase:              Succeeded
Start:              2022-06-04 18:37:07 -0500 CDT
Finished:           2022-06-04 18:37:08 -0500 CDT
Duration:           1s
Message:            successfully synced (all tasks run)

GROUP  KIND        NAMESPACE  NAME          STATUS  HEALTH   HOOK  MESSAGE
       Service     default    guestbook-ui  Synced  Healthy        service/guestbook-ui created
apps   Deployment  default    guestbook-ui  Synced  Healthy        deployment.apps/guestbook-ui created
```

Y volvemos a listar la aplicación:

```shell
$ argocd app get guestbook
Name:               guestbook
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://cd.apis.example.io/applications/guestbook
Repo:               https://github.com/argoproj/argocd-example-apps.git
Target:
Path:               guestbook
SyncWindow:         Sync Allowed
Sync Policy:        <none>
Sync Status:        Synced to  (53e28ff)
Health Status:      Healthy

GROUP  KIND        NAMESPACE  NAME          STATUS  HEALTH   HOOK  MESSAGE
       Service     default    guestbook-ui  Synced  Healthy        service/guestbook-ui created
apps   Deployment  default    guestbook-ui  Synced  Healthy        deployment.apps/guestbook-ui created
```

Ahora ya podemos borrar la aplicación:

```shell
$ argocd app delete guestbook
Are you sure you want to delete 'guestbook' and all its resources? [y/n]
y
```

### Configuración de repositorio Git

Entramos al UI, vamos al menú `Settings`, `Repositories` y hacemos clic en `CONNECT REPO USING SSH`, y definimos los
siguientes parámetros:

 * Name: gcloud-kronops-cloud-configs
 * Project: default
 * Repository URL: ssh://boo.bar@kronops.mx@source.developers.google.com:2022/p/kronops-dev/r/kronops-cloud-configs
 * SSH private key data: ******
 * Skip server verification: enabled

### Agregando Aplicaciones

Las aplicaciones son definidas por ambiente, por lo que para dar de alta la aplicación `whoami` en el ambiente
`sandbox`, vamos al menú `Applications` y hacemos clic en `New App`:

En la sección `General` usamos:

 * `Application Name:` whoami-sandbox 
 * `Project:`  Default
 * `Sync Policy:` Manual
 
En la sección `Source` usamos:

 * `Repository URL:` ssh://foo.bar@kronops.mx@source.developers.google.com:2022/p/kronops-dev/r/kronops-cloud-configs
 * `Revision:` HEAD 
 * `Path:` sandbox/whoami

En la sección `Destination` usamos:

 * `Cluster URL:` https://kubernetes.default.svc
 * `Namespace:`  whoami
 
Si las configuraciones son correctos se agrega la aplicación y por default aparece en estado `OutOfSync`, ahora
seleccionamos la aplicación y hacemos clic en `APP DETAILS`, después en `EDIT` y modificamos `LABELS` con los
siguientes datos:

 * proyect=aaap
 * environment=sandbox
 * team=octopods
 * product-owner=jyr-gaxiola

Terminamos de editar las etiquetas y hacemos clic en `Save`.

Lo siguiente es hacer clic en el botón `SYNC` para sincronizar el cluster con el repositorio Git.

Opcionalmente, puede realizar la misma operación de forma declarativa, creamos un archivo `YAML`:

```shell
$ vim whoami.yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: whoami-sandbox
  namespace: argocd
  labels:
    proyect: aaap
    environment: sandbox
    team: octopods
    product-owner: jyr-gaxiola
spec:
  destination:
    namespace: whoami
    server: https://kubernetes.default.svc
  project: contratos
  source:
    path: sandbox/whoami
    repoURL: ssh://foo.bar@kronops.mx@source.developers.google.com:2022/p/kronops-dev/r/kronops-cloud-configs
    targetRevision: HEAD
```

Ahora agregamos la aplicación a Argo CD:

```shell
$ k apply -f whoami.yml
application.argoproj.io/whoami-sandbox created
```

Ahora listamos la aplicación:

```shell
$ argocd app list
NAME            CLUSTER                         NAMESPACE  PROJECT    STATUS  HEALTH   SYNCPOLICY  CONDITIONS  REPO                                                                                                              PATH            TARGET
whoami-sandbox  https://kubernetes.default.svc  default     contratos  Synced  Healthy  <none>      <none>      ssh://foo.bar@kronops.mx@source.developers.google.com:2022/p/kronops-dev/r/kronops-cloud-configs  sandbox/whoami  HEAD
```

Finalmente sincronizamos la aplicación:

```shell
$ argocd app sync whoami-sandbox
```

### Respaldo y Recuperación

Con la herramienta cli `argocd` es posible exportar la configuración a un archivo yaml, por ejemplo:

```shell
$ argocd -n argocd admin export > argocd-2022-04-21.yml
```

### Image Updater
 
Instalamos en el cluster image updater:

```shell
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
serviceaccount/argocd-image-updater created
role.rbac.authorization.k8s.io/argocd-image-updater created
rolebinding.rbac.authorization.k8s.io/argocd-image-updater created
configmap/argocd-image-updater-config created
configmap/argocd-image-updater-ssh-config created
secret/argocd-image-updater-secret created
deployment.apps/argocd-image-updater created
```

Como se puede ver se crearon diferentes recursos de kubernetes, cuentas de servicio, roles, rolebinding,
configmaps, secretos y el despliegue.

Listamos los pods en el namespace:

```shell
$ k -n argocd get pods
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          27d
argocd-applicationset-controller-76cb76dfb5-2dzhb   1/1     Running   0          27d
argocd-dex-server-754c947c74-8x782                  1/1     Running   0          27d
argocd-image-updater-745c984bdd-s7d2h               1/1     Running   0          39s
argocd-notifications-controller-548b59d47c-kb99t    1/1     Running   0          27d
argocd-redis-86c97c8c64-qq22g                       1/1     Running   0          27d
argocd-repo-server-766748bdd7-srtpx                 1/1     Running   0          27d
argocd-server-66dd8b54b6-86q68                      1/1     Running   0          27d
```

Como se puede ver ya está corriendo el pod de `argocd-image-updater`, y tiene estado `Running`.

### Referencias

 * [Argo CD - Getting Started](https://argo-cd.readthedocs.io/en/stable/getting_started/)
 * [Argo CD Image Updater - Getting Started](https://argocd-image-updater.readthedocs.io/en/stable/install/installation/)
