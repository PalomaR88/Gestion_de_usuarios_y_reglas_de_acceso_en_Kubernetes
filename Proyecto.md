# Gestión de usuarios y reglas de acceso en Kubernetes
# 1. Introducción

Kubernetes es el orquestador de contenedores más utilizado en la actualidad. Una de las claves de su éxito es la configuración que ofrece. Y un punto fundamental para cualquier administrador de sitemas pasa por saber administrar los usuarios que utilizarán nuestro cluster. 

El presente proyecto integrado trata la gestión de los usuarios en Kubernetes y la aplicación de permisos a los mismo, para poder administrar correctamente un cluster.

El documento se divide en dos partes principales. La primera sobre el control de la API, se divide a su vez en **autenticación** donde se explica algunas de las diversas formas que Kubernetes utiliza para autenticar a los usuario. **Autorización** se explica cómo Kubernetes permite a un usuario formar parte del cluster. Por último, se explica la concesión de permisos a través de **RBAC** y el control de los recursos a través de **Quotas**.

La segunda parte del documento aborda algunas herramientas complementarias que pueden ser de utilidad para la gestión de usuarios.

Para ilustrar todas estas ideas las partes del trabajo se componen de una introducción teórica donde se explican los conceptos más importantes y un caso práctico que ayudará a consolidar la parte teórica y comprenderla.

# 2. Creación del escenario y recomendaciones

Para los casos prácticos se ha utilizado un cluster de Kubernetes, instalado sobre tres máquinas con sistemas operativos Debian Buster y una máquina cliente sobre el mismo sistema. Las caracterísitca de las máquinas son las siguientes:

* **kubemaster**: Máquina master del cluster de Kubernetes. 
  * 4GB de RAM.
  * 40GB de disco.

* **kubeminion1** y **kubeminion2**: Máquinas minions del cluster de Kubernetes. 
  * 2GB de RAM.
  * 20GB de disco.

* **kubecliente**: Máquina cliente del cluster de kubernetes. 
  * 1GB de RAM.
  * 20GB de disco.


Para la creación de este escenario se ha seguido una [guía propia de instalación](https://github.com/PalomaR88/kubernete/blob/master/Practica.md#instalaci%C3%B3n-de-kubernetes).

Los puertos necesarios que vamos a abrir para este proyecto son los siguientes:


|Puerto|Descripción|
|------|----------------------|
|6443  |Puerto de acceso a la API en un cluster de Kubernetes|
|80| Puerto de acceso a los servicios de http
|443| Puerto de acceso a los servicios de https
|30000-40000| Puerto de aplicaciones con el servicio NodePort


# 3. Control de acceso a la API
Para que los usuarios puedan ser autorizados para el acceso a la API se pasa por 3 etapas: 

1. Autenticación
2. Autorización
3. Control de admisión.

Los clúster de Kubernetes tienen un certificado, que suele estar autorfimado, y se incluyen en `$USER/.kube/config`. Este cerficado se escribe automáticamente cuando se crea un clúster y se puede compartir con otros usuarios.

## 3.1. Autenticación

Hay dos categorías de usuarios en Kubernetes: cuentas administradas por Kubernetes y usuarios normales administrados por un servicio externo e independiente como Keystone, Google Accounts o una lista de ficheros.

Aquí se trabajará con las cuentas administradas por Kubernetes. Se crean automáticamente por el servidor API o manualmente a través de una llamada. Éstas están vinculadas a un conjunto de credenciales almacenadas como Secrets.

Para autenticar las solicitudes de API Kubernetes utiliza certificados de cliente, tokens, proxy de autenticación o autenticación básica HTTP. Para las solicitud se utilizan los siguientes atributos: nombre de usuario, UID, grupos y campos adicionales.

Se pueden habilitar varios métodos de autenticación a la vez.

### 3.1.1. Certificados de cliente X509
Se debe generar certificados desde la línea de comandos a través de **easyrsa**, **openssl** o **cfssl**. En esta tarea se explica la creación de certificados a través de **openssl**. 

Estos certificados deben ser firmados por la clave y el certificado que proporciona Kubernetes. En el caso de tener instalado kubeadm, éstos, se encuentran en `/etc/kubernetes/pki/` y en minikube en `~/.minikube/`. También se pueden crear credenciales propias, para ello se necesita una entidad certificadora. 

**Creación del certificado del cliente**

En primer lugar, el usuario tiene que generar la clave y la petición de firma.

1. Generación de la clave.
~~~
openssl genrsa -out <nombre_key>.key 2048
~~~

2. Generación de la petición de firma.
~~~
openssl req -new -key <nombre_key>.key -out <nombre_csr>.csr -subj "/CN=<commonName_del_usuario>/O=<grupo>"
~~~

A continuación, un usuario con permisos de administrador del clúster firma y crea el certificado.

3. Creación del certificado.
~~~
openssl x509 -req -in <nombre_csr>.csr -CA /<ruta_certificado_CA>/ca.crt -CAkey /<ruta_key_CA>/ca.key -CAcreateserial -out <nombre_certificado>.crt [-days <nº_días>]
~~~

4. Creación de los usuarios en el clúster (desde el administrador del clúster).
~~~
kubectl config set-credentials <usuario> --client-certificate=<nombre_certificado>.crt --client-key=<nombre_clave>.key
~~~

Para comprobar los usuarios que se han creado:
~~~
kubectl config view
~~~

#### 3.1.1.1. Caso práctico de autenticación con certificado
Se genera la clave y la petición de firma:
~~~
debian@kubecliente:~$ openssl genrsa -out kubecliente.key 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
....................+++++
................+++++
e is 65537 (0x010001)
debian@kubecliente:~$ openssl req -new -key kubecliente.key -out kubecliente.csr -subj "/CN=kubecliente/O=proyecto"
~~~

Se envía la petición de firma al nodo master para que se cree el certificado:
~~~
debian@kubecliente:~$ export IP_MASTER=172.22.200.133
debian@kubecliente:~$ scp kubecliente.csr debian@${IP_MASTER}:
debian@172.22.200.133's password: 
kubecliente.csr                                  100%  920   554.1KB/s   00:00 
~~~

Desde el nodo master se firma la petición:
~~~
debian@kubemaster:~$ sudo openssl x509 -req -in kubecliente.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out kubecliente.crt -days 90
Signature ok
subject=CN = kubecliente, O = proyecto
Getting CA Private Key
~~~

A continuación, el cliente obtiene el certificado:
~~~
debian@kubecliente:~$ sftp debian@${IP_MASTER}
debian@172.22.200.133's password: 
Connected to debian@172.22.200.133.
sftp> get /home/debian/k
kubecliente.crt     kubecliente.csr     
sftp> get /home/debian/kubecliente.crt 
Fetching /home/debian/kubecliente.crt to kubecliente.crt
/home/debian/kubecliente.crt                     100% 1021   302.5KB/s   00:00    
sftp> exit
~~~

Se crea el usuario en el clúster:
~~~
debian@kubemaster:~$ kubectl config set-credentials kubecliente --client-certificate=kubecliente.crt --client-key=kubecliente.key
User "kubecliente" set.
~~~

Y se comprueba que el usuario se ha creado correctamente:
~~~
debian@kubemaster:~$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://10.0.0.3:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubecliente
  user:
    client-certificate: /home/debian/kubecliente.crt
    client-key: /home/debian/kubecliente.key
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
~~~


## 3.2. Autorización
Para ejecutar comandos con un usuario concreto se utilizan los **contextos**. Para ver los contextos existentes se utiliza la siguiente orden:
~~~
kubectl config get-contexts  
~~~

Para crear un contexto:
~~~
kubectl config set-context <contexto> --cluster=<nombre_cluster> --user=<usuario>
~~~

Para cambiar el contexto:
~~~
kubectl config use-context <contexto>
~~~

Y para ver el contexto que se está usando:
~~~
kubectl config current-context
~~~

### 3.2.1. Caso práctico de autorización

A continuación, se va a crear un contexto para el nodo y usuario cliente:
~~~
debian@kubemaster:~$ kubectl config set-context kubecliente --cluster=kubernetes --user=kubecliente
Context "kubecliente" created.
~~~

Y se comprueba que se ha creado:
~~~
debian@kubemaster:~/RBAC$ kubectl config get-contexts
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
          kubecliente                   kubernetes   kubecliente        
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin 
~~~

Se instala kubectl en el nodo cliente y desde el master se dan permiso de lectura al fichero `/etc/kubernetes/admin.conf`:
~~~
debian@kubemaster:~$ sudo chmod 644 /etc/kubernetes/admin.conf
~~~

Desde el nodo cliente se va copiar el fichero de configuración del master:
~~~
debian@kubecliente:~$ sftp debian@${IP_MASTER}
The authenticity of host '172.22.200.133 (172.22.200.133)' can't be established.
ECDSA key fingerprint is SHA256:Y+knQJVp5El7mt7x/P3yI74ZhoAi2AF9fwIDsMEbhtU.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '172.22.200.133' (ECDSA) to the list of known hosts.
debian@172.22.200.133's password: 
Connected to debian@172.22.200.133.
sftp> get /etc/kubernetes/admin.conf
Fetching /etc/kubernetes/admin.conf to admin.conf
/etc/kubernetes/admin.conf                       100% 5444     1.4MB/s   00:00    
sftp> exit
~~~

Y se crea el directorio `.kube` para alojar el fichero de configuración:
~~~
debian@kubecliente:~$ mkdir .kube
debian@kubecliente:~$ mv admin.conf ~/.kube/mycluster.conf
debian@kubecliente:~$ sed -i -e "s#server: https://.*:6443#server: https://${IP_MASTER}:6443#g" ~/.kube/mycluster.conf
debian@kubecliente:~$ export KUBECONFIG=~/.kube/mycluster.conf
~~~

Finalmente se comprueba que funciona correctamente:
~~~
debian@kubecliente:~$ kubectl cluster-info
Kubernetes master is running at https://172.22.200.133:6443
KubeDNS is running at https://172.22.200.133:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
~~~

Se cambia el contexto:
~~~
debian@kubecliente:~$ kubectl config use-context kubecliente
Switched to context "kubecliente".
~~~

Y se comprueba que se ha cambiado de contexto correctamente:
~~~
debian@kubecliente:~$ kubectl config current-context
kubecliente
~~~


## 3.3. Autorización RBAC y Quotas
El **control de acceso basado en roles** (RBAC) es un método para regular el acceso a los recursos en función de los roles de los usuarios. Para ello utiliza **rbac.authorization.k8s.io** que permite configurar dinámicamente políticas a través de la API de Kubernetes. 

### 3.3.1. Objetos API
La API RBAC declara cuatro tipos de objetos: Role, ClusterRole, RoleBinding y ClusterRoleBinding. Para trabajar con estos objetos, como todos los objetos de Kubernetes, se debe usar la API de Kubernetes. 

Los objetos de Kubernetes son entidades persistentes en el sistema de Kubernetes que se usan para representar el estado del cluster. Pueden describir:

- Qué aplicaciones en contenedores se están ejecutando y en qué nodos.
- Los recursos disponibles para esas aplicaciones. 
- Las políticas sobre cómo se comportan esas aplicaciones (renicio, actualizaciones, tolerancia a fallos, etc.).

#### 3.3.1.1. Role y ClusterRole
Un Role RBAC o ClusterRole contiene reglas que representan un conjunto de permisos siendo estos aditivos, es decir, no se usan reglas de negación. 

En el caso de Role RBAC se establecen permisos dentro de un namespace particular, mientras que los ClusterRole permite definir un mismo rol en todo el cluster.

Ejemplo de Role:
~~~
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
    name: <nombre_rol>
    namespace: <namespace>
rules:
- apiGroups: [""]
  resources: [""]
  verbs: [""]
...
~~~

Ejemplo de ClusterRole:
~~~
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
    name: <nombre_rol>
rules:
- apiGroups: [""]
  resources: [""]
  verbs: [""]
...
~~~

Los dos atributos claves para entender la sección **rules** son los recursos y las acciones, es decir, **Resources** y **Verbs**

Los recursos, **resources**, como se ha indicado anteriormente, son los conjuntos de objetos API de Kubernetes disponibles en el cluster. Estos son algunos de los recursos: `"pods"`, `"secrets"`, `"deployments"`, `"services"`, `"configmaps"`, `"endpoints"`, `"crontrabs"`, `"jobs"`, `"nodes"`, `"replicasets"`, `"configMaps"` y `"autoScaler"`. Para indicar subrecursos se emplea la barra inclinada, por ejemplo, el subrecurso log de los pods se indica de la siguiente forma: `"pods/log"`.

Con la opción **resourceNames** se puede consultar los recursos por nombre. Esta opción no se puede usar con las restricciones `create` o `deletecollection` por razones lógicas.

El conjunto de operaciones que se pueden ejecutar sobre los recursos se indican en **verbs**. Son, por ejemplo, `"get"`, `"watch"`, `"create"`, `"delete"`, `"list"`, `"patch"` y `"update"`.

A continuación, se van a desarrollar algunos ejemplos de la sección rules para poner en práctica lo anteriormente explicado:

- Permitir que se puedan listar los pods, es decir, ejecutar el comando `get pods`:
~~~
rules:
 - apiGroups: ["*"]
   resources: ["pods"]
   verbs: ["list"]
~~~

- Permitir acceso de solo lectura en los pods, sin poder eliminar pods directamente pero sí a través de implementaciones, **deployments**:
~~~
 rules:
 - apiGroups: ["*"]
   resources: ["pods"]
   verbs: ["list","get","watch"]
 - apiGroups: ["extensions","apps"]
   resources: ["deployments"]
   verbs: ["get","list","watch","create","update","patch","delete"]
~~~

#### 3.3.1.2. RoleBinding y ClusterRoleBinding

RoleBinding o ClusterRoleBinding otorga los permisos definidos en un rol a un usuario o cuentas de servicios, **ServiceAccount**. RoleBinding otorga permisos dentro de un namespace específico, mientras que un ClusterRoleBinding otorga acceso a todo el cluster.

Un RoleBinding puede hace referencia a un Role, en el mismo namespace, o a un ClusterRole. En este caso, el ClusterRole se vincula al namespace específico del RoleBinding. Si se quiere vincular un ClusterRole en todo el cluster se debe utilizar ClusterRoleBinding.

Ejemplo de RoleBinding:
~~~
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: <nombre_enlace_rol>
  namespace: <namespace>
subjects:
- kind: [ User | Group | ServiceAccount ]
  name: <nombre_usuario/grupo/ServiceAccount>
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: [ Role | ClusterRole ]
  name: <nombre_Role/ClusterRole>
  apiGroup: rbac.authorization.k8s.io
~~~

Ejemplo de ClusterRoleBinding:
~~~
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: <nombre_enlace_rol>
  namespace: <namespace>
subjects:
- kind: [ User | Group | ServiceAccount]
  name: <nombre_usuario/grupo/ServiceAccount>
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: [ Role | ClusterRole ]
  name: <nombre_Role/ClusterRole>
  apiGroup: rbac.authorization.k8s.io
~~~

### 2.3.2. Cuotas de recursos
Las cuotas de recursos es una herramienta para administrar los recursos del clúster.

Para ver las cuotas disponibles en un namespace:
~~~
kubectl describe quota -n <namespace>
~~~

Para ver la información detallada de las cuotas:
~~~
kubectl get resourcequota <nombre_cuota> --namespace=<namespace> --output=yaml
~~~


Un ejemplo de la sintaxis del fichero de configuración de las cuotas es la siguiente:
~~~
apiVersion: v1
kind: ResourceQuota
metadata:
  name: <nombre-cuota>
spec:
  hard:
    [pods: "<nº>"]
    [requests.cpu: "<nº>"]
    [requests.memory: <nº>Gi]
    [limits.cpu: "<nº>"]
    [limits.memory: <nº>Gi]
    [persistentvolumeclaims: "<nº>"]
    [configmaps: "<nº>"]
    [replicationcontrollers: "<nº>"]
    [secrets: "<nº>"]
    [services: "<nº>"]
    [services.loadbalancers: "<nº>"]
...
~~~

### 3.3.3. Caso práctico de autorización RBAC y Quotas

Para este caso práctico vamos a continuar con el contexto y usuario que se ha creado anteriormente: kubecliente. Con este usuario se va a desplegar una aplicación Wordpress, con una base de datos, sobre volúmenes persistentes. 

Con este usuario no se puede hacer nada, ya que no tiene permisos para nada, ni en su propio namespace:
~~~
debian@kubecliente:~$ kubectl get pods -n kubecliente
Error from server (Forbidden): pods is forbidden: User "kubecliente" cannot list resource "pods" in API group "" in the namespace "kubecliente"
~~~

Se van a establecer unas normas para que el usuario pueda utilizar este espacio de nombres, es decir, se va a crear un rol a través de un fichero de configuración que va a permitir, en el namespace correspondiente, unas funciones básicas, que son: ver y listar pods y, además se puede ver, listar, crear, actualizar, borrar, etc. los despliegues.
~~~
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
    name: despliegue-kubecliente
    namespace: kubecliente
rules:
 - apiGroups: ["*"]
   resources: ["pods"]
   verbs: ["list","get","watch"]
 - apiGroups: ["extensions","apps"]
   resources: ["deployments"]
   verbs: ["get","list","watch","create","update","patch","delete"]
~~~

Y se aplica el rol:
~~~
debian@kubemaster:~/RBAC$ kubectl apply -f rol-despliegue-kubecliente.yaml 
role.rbac.authorization.k8s.io/despliegue-kubecliente created
~~~

De esta forma se comprueba que se ha creado el rol en el namespace kubecliente:
~~~
debian@kubemaster:~/RBAC$ kubectl get role -n kubecliente
NAME                     CREATED AT
despliegue-kubecliente   2020-05-25T15:58:06Z
~~~

Tras la creación del rol se asigna a un usuario, para ello se crea un objeto RoleBinding, configurado en un fichero con extensión YAML con la siguiente sintaxis:
~~~
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: RBdespliegues-kubecliente
  namespace: kubecliente
subjects:
- kind: User
  name: kubecliente
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: despliegue-kubecliente
  apiGroup: rbac.authorization.k8s.io
~~~

Y se aplica:
~~~
debian@kubemaster:~/RBAC$ kubectl apply -f despliegue-kubecliente.yaml 
rolebinding.rbac.authorization.k8s.io/RBdespliegues-kubecliente created
~~~

Por último, se comprueba que el cliente puede ver los pods de su correspondiente espacio de nombres:
~~~
debian@kubecliente:~$ kubectl get pods -n kubecliente
No resources found in kubecliente namespace.
~~~

El siguiente paso será crear un servicio tipo **ClusterIP**, que posteriormente conectará el pod encargado de la base de datos de MariaDB con el pod que alojará Wordpress.

Para ello se crea un fichero `.yaml` con el siguiente contenido:
~~~
apiVersion: v1
kind: Service
metadata:
  name: mariadb-service
  namespace: kubecliente
  labels:
    app: wordpress
    type: database
spec:
  selector:
    app: wordpress
    type: database
  ports:
  - port: 3306
    targetPort: db-port
  type: ClusterIP
~~~

Y se intenta crear el servicio:
~~~
debian@kubecliente:~/desp-wp$ kubectl create -f serv-clusterip.yaml 
Error from server (Forbidden): error when creating "serv-clusterip.yaml": services is forbidden: User "kubecliente" cannot create resource "services" in API group "" in the namespace "kubecliente": RBAC: role.rbac.authorization.k8s.io "rol-despliegue-kubecliente" not found
~~~

El mensaje que aparece indica que este usuario no puede crear servicios en este espacio de nombres. Por lo tanto, a continuación, se van a crear dos reglas, una para ver los servicios y otra para crearlos. Estas dos reglas podrían crearse a la par, pero aquí la vamos a desgranar en dos roles.

En primer lugar, desde el master, se creará un rol que permita ver los servicios que se han creado:
~~~
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: get-serv-kubecliente
  namespace: kubecliente
rules:
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "watch"]
~~~

Y a través de otro fichero se asigna este nuevo rol al usuario:
~~~
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: RBget-serv-kubecliente
  namespace: kubecliente
subjects:
- kind: User
  name: kubecliente
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: get-serv-kubecliente
  apiGroup: rbac.authorization.k8s.io
~~~

Tras crear el rol y aplicar el RoleBinding se comprueba que ahora sí se pueden ver los servicios:
~~~
debian@kubecliente:~/desp-wp$ kubectl get services -n kubecliente
No resources found in kubecliente namespace.
~~~

Lo siguiente será otorgarle los permisos para crear servicios en este namespace. Para ello, nuevamente se creará un rol con la siguiente configuración:
~~~
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: crear-serv-kubecliente
  namespace: kubecliente
rules:
- apiGroups: [""]
  resources: ["services"]
  verbs: ["create","update","patch","delete"]
~~~

Y, el correspondiente fichero, para aplicar la regla al usuario:
~~~
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: RBcrear-serv-kubecliente
  namespace: kubecliente
subjects:
- kind: User
  name: kubecliente
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: crear-serv-kubecliente
  apiGroup: rbac.authorization.k8s.io
~~~

Con estos roles creados y asignados, se vuelve a crear el servicio desde el cliente y se comprueba que se ha creado correctamente:
~~~
debian@kubecliente:~/desp-wp$ kubectl create -f serv-clusterip.yaml 
service/mariadb-service created
debian@kubecliente:~/desp-wp$ kubectl get services -n kubecliente
NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
mariadb-service   ClusterIP   10.101.185.102   <none>        3306/TCP   64s
~~~

Lo siguiente será crear un fichero donde se indican los datos para MariaDB que se guardarán en un secreto:
~~~
apiVersion: v1
data:
  dbname: ZGJfd29yZHByZXNz
  dbpassword: ZGJfcGFzcw==
  dbrootpassword: ZGJfcm9vdA==
  dbuser: d3BfdXNlcg==
kind: Secret
metadata:
  creationTimestamp: null
  name: mariadb-secret
  namespace: kubecliente
~~~

Al implementar este fichero aparece el siguiente error que indica que el usuario no tiene permisos para crear secretos:
~~~
debian@kubecliente:~/desp-wp$ kubectl create -f secret-mariadb.yaml 
Error from server (Forbidden): error when creating "secret-mariadb.yaml": secrets is forbidden: User "kubecliente" cannot create resource "secrets" in API group "" in the namespace "kubecliente": RBAC: role.rbac.authorization.k8s.io "rol-despliegue-kubecliente" not found
~~~

A continuación, se creará un rol desde kubemaster que permita al usuario ver, crear, borrar, etc. secretos en el namespace kubecliente:
~~~
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
    name: all-secrets-kubecliente
    namespace: kubecliente
rules:
 - apiGroups: [""]
   resources: ["secrets"]
   verbs: ["get","list","watch","create","update","patch","delete"]
~~~

Y se añade el rol al usuario:
~~~
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: RBaal-secrets-kubecliente
  namespace: kubecliente
subjects:
- kind: User
  name: kubecliente
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: all-secrets-kubecliente
  apiGroup: rbac.authorization.k8s.io
~~~

Y desde el cliente ya te permite crear el secreto:
~~~
debian@kubecliente:~/desp-wp$ kubectl create -f secret-mariadb.yaml 
secret/mariadb-secret created
debian@kubecliente:~/desp-wp$ kubectl get secrets -n kubecliente
NAME                  TYPE                                  DATA   AGE
default-token-sqh67   kubernetes.io/service-account-token   3      2d1h
mariadb-secret        Opaque                                4      3s
~~~

El siguiente fichero crea un servicio **NodePort**, es decir, que mapea el puerto 80 del contenedor a un puerto entre el 30000 y el 40000. Como se va a crear un servicio y este usuario tiene permisos para ello, Kubernetes permite realizar esta acción :
~~~
apiVersion: v1
kind: Service
metadata:
  name: wordpress-service
  namespace: kubecliente
  labels:
    app: wordpress
    type: frontend
spec:
  selector:
    app: wordpress
    type: frontend
  ports:
    - name: http-sv-port
      port: 80
      targetPort: http-port
    - name: https-sv-port
      port: 443
      targetPort: https-port
  type: NodePort
~~~

Este despliegue utiliza dos volúmenes persistentes que han sido creados con anterioridad por kubemaster. Para la creación de estos volúmenes se ha seguido una [guía propia](https://github.com/PalomaR88/Volumenes-persistentes-kubernetes/blob/master/Practica.md).

Además, se va a suminstrar almacenamiento dinámico desde el administrador al cliente para que pueda crear llamadas a los volúmenes asignados libremente. Para ello se crean dos **StorageClass**. Uno por defecto, el usuario no podrá hacer uso de los volúmenes mascardo con este tipo. Y el storegeclass que más tarde se le asignará al usuario, vol-kubecliente. El fichero de configuración es el siguiente:
~~~ 
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: vol-kubecliente
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
---
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: default
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
~~~

Es necesario indicar el storageclass por defecto, cambiando el valor de `storageclass.kubernetes.io/is-default-class` por `true`:
~~~
debian@kubemaster:~/RBAC$ kubectl patch storageclass default -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}' -n kubecliente
storageclass.storage.k8s.io/default patched
~~~

Si se quiere cambiar de clase por defecto, en el caso de existir ya una, se debe modificar `true` por `false`.

Finalmente, el resultado es que hay dos storageclass, uno de ellos por defecto y el que se le asignará al usuario:
~~~
debian@kubemaster:~/RBAC$ kubectl get storageclass
NAME                PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
default (default)   kubernetes.io/gce-pd   Delete          Immediate           false                  91s
vol-kubecliente     kubernetes.io/gce-pd   Delete          Immediate           false                  5h36m
~~~

A continuación, se van a crear los dos volúmenes persistentes necesarios. A modo de prueba, se indicará en el fichero de configuración que uno de los volúmenes es de tipo default y el otro vol-kubecliente para comprobar más tarde que el usuario no puede hacer uso del volúmen marcado por defecto:
~~~
apiVersion: v1
kind: PersistentVolume
metadata:
  name: vol1
spec:
  storageClassName: vol-kubecliente
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /home/shared/volumen1
    server: 10.0.0.3
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: vol2
spec:
  storageClassName: default
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /home/shared/volumen2
    server: 10.0.0.3
~~~

El resultado es el siguiente:
~~~
debian@kubemaster:~/RBAC$ kubectl get pv
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                       STORAGECLASS      REASON   AGE
vol1   5Gi        RWX            Recycle          Bound       kubecliente/clientewp-pvc   vol-kubecliente            6h40m
vol2   5Gi        RWX            Recycle          Available                               default                    6h40m
~~~

El control del uso de los volúmenes a través de storageclass se va a realizar a través de una quota donde se indica, a través de `<storageclass>.storageclass.storage.k8s.io/persistentvolumeclaims: <numero>` la cantidad de llamadas que se pueden hacer a las clases:
~~~
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-storageclass-kubecliente
spec:
  hard:
    default.storageclass.storage.k8s.io/persistentvolumeclaims: 0
    vol-kubecliente.storageclass.storage.k8s.io/persistentvolumeclaims: 10
~~~

Y se aplica la quota en el namespace:
~~~
debian@kubemaster:~/RBAC$ kubectl create -f quota.yaml -n kubecliente
resourcequota/object-quota-demo created
~~~

Pero de momento, el usuario sigue sin tener permiso para crear llamadas, por lo que es necesario el siguiente rol y rolebinding:
~~~
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
    name: all-PVClaim-kubecliente
    namespace: kubecliente
rules:
 - apiGroups: [""]
   resources: ["persistentvolumeclaims"]
   verbs: ["get","list","watch","create","update","delete"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: RBall-PVClaim-kubecliente
  namespace: kubecliente
subjects:
- kind: User
  name: kubecliente
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: all-PVClaim-kubecliente
  apiGroup: rbac.authorization.k8s.io
~~~

Ahora sí, con los permisos oportunos, desde el cliente se van a crear dos peticiones de llamadas con el siguiente fichero:
~~~
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: clientewp-pvc
  namespace: kubecliente
spec:
  storageClassName: vol-kubecliente
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb-pvc
  namespace: kubecliente
spec:
  storageClassName: vol-kubecliente
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
~~~

Se debe utilizar una clase válida, es decir, si en la etiqueta `storageClassName` se indica `default` aparece el siguiente error:
~~~
debian@kubecliente:~/desp-wp$ kubectl create -f PVC.yaml 
Error from server (Forbidden): error when creating "PVC.yaml": persistentvolumeclaims "clientewp-pvc" is forbidden: exceeded quota: object-quota-demo, requested: default.storageclass.storage.k8s.io/persistentvolumeclaims=1, used: default.storageclass.storage.k8s.io/persistentvolumeclaims=0, limited: default.storageclass.storage.k8s.io/persistentvolumeclaims=0
~~~

Una vez creados los llamamientos, se comprueba como uno de ellos está pendiente:
~~~
debian@kubecliente:~/desp-wp$ kubectl get persistentvolumeclaims -n kubecliente
NAME            STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS      AGE
clientewp-pvc   Pending                                      vol-kubecliente   36s
mariadb-pvc     Bound     vol1     5Gi        RWX            vol-kubecliente   40s
~~~

Esto ocurre porque no hay suficiente almacenamiento del tipo vol-kubecliente. Para obtener más información sobre el persistentvolumeclaims que se encuentra en este estado se puede utilizar el comando `kubectl describe persistentvolumeclaim <nombre_persistentvolumeclaims>`.

Para solucionarlo, desde el máster, se modifica el fichero de configuración que se ha usado para la creación de los volúmenes persistentes, siendo ahora ambos volúmenes del tipo vol-kubecliente:
~~~
...
apiVersion: v1
kind: PersistentVolume
metadata:
  name: vol2
spec:
  storageClassName: vol-kubecliente
...
~~~

Se modifica:
~~~
debian@kubemaster:~/RBAC$ kubectl apply -f vol-persistente.yaml 
persistentvolume/vol1 unchanged
persistentvolume/vol2 configured
~~~

Y comprueba el cambio de estado:
~~~
debian@kubecliente:~/desp-wp$ kubectl get persistentvolumeclaims -n kubecliente
NAME            STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS      AGE
clientewp-pvc   Bound    vol2     5Gi        RWX            vol-kubecliente   2m15s
mariadb-pvc     Bound    vol1     5Gi        RWX            vol-kubecliente   2m19s
~~~

Aunque no es necesario para esta práctica, en el caso de otorgarle al usuario permisos para ver los volúmenes, se realiza a través de un clusterRole y ClusterRoleBinding:
~~~
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: almacenamiento-kubecliente
rules:
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get","list","watch","update"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: CRBalmacenamiento-kubecliente
roleRef:
  kind: ClusterRole
  name: almacenamiento-kubecliente
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: User
  name: kubecliente
  apiGroup: rbac.authorization.k8s.io
~~~


A continuación, se despliega mariadb definiendo el volumen:
~~~
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb-deployment
  namespace: kubecliente
  labels:
    app: wordpress
    type: database
spec:
  selector:
    matchLabels:
      app: wordpress
  replicas: 1
  template:
    metadata:
      labels:
        app: wordpress
        type: database
    spec:
      containers:
        - name: wordpress
          image: mariadb
          ports:
            - containerPort: 3306
              name: db-port
          env:
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mariadb-secret
                  key: dbuser
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: mariadb-secret
                  key: dbname
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mariadb-secret
                  key: dbpassword
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mariadb-secret
                  key: dbrootpassword
          volumeMounts: 
            - name: vol1
              mountPath: /var/lib/mysql
      volumes:
        - name: vol1
          persistentVolumeClaim:
            claimName: mariadb-pvc
~~~

Y la creación del deployment de wordpress definiendo el volumen:
~~~
apiVersion: apps/v1
kind: Deployment
metadata:
  name: despliegue-wp
  namespace: kubecliente
  labels:
    app: wordpress
    type: frontend
spec:
  selector:
    matchLabels:
      app: wordpress
  replicas: 1
  template:
    metadata:
      labels:
        app: wordpress
        type: frontend
    spec:
      containers:
        - name: wordpress
          image: wordpress
          ports:
            - containerPort: 80
              name: http-port
            - containerPort: 443
              name: https-port
          env:
            - name: WORDPRESS_DB_HOST
              value: mariadb-service
            - name: WORDPRESS_DB_USER
              valueFrom:
                secretKeyRef:
                  name: mariadb-secret
                  key: dbuser
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mariadb-secret
                  key: dbpassword
            - name: WORDPRESS_DB_NAME
              valueFrom:
                secretKeyRef:
                  name: mariadb-secret
                  key: dbname
          volumeMounts:
            - name: vol2
              mountPath: /var/www/html
      volumes:
        - name: vol2
          persistentVolumeClaim:
            claimName: clientewp-pvc
~~~

Este es el resultado:
~~~
debian@kubecliente:~/desp-wp$ kubectl get deployment,service,pv,pvc,pods -n kubecliente
NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/despliegue-wp        1/1     1            1           6m23s
deployment.apps/mariadb-deployment   1/1     1            1           7m11s

NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/mariadb-service     ClusterIP   10.100.157.197   <none>        3306/TCP                     2d23h
service/wordpress-service   NodePort    10.102.158.19    <none>        80:31358/TCP,443:32216/TCP   8h

NAME                    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                       STORAGECLASS      REASON   AGE
persistentvolume/vol1   5Gi        RWX            Recycle          Bound    kubecliente/mariadb-pvc     vol-kubecliente            8h
persistentvolume/vol2   5Gi        RWX            Recycle          Bound    kubecliente/clientewp-pvc   vol-kubecliente            8h

NAME                                  STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS      AGE
persistentvolumeclaim/clientewp-pvc   Bound    vol2     5Gi        RWX            vol-kubecliente   80m
persistentvolumeclaim/mariadb-pvc     Bound    vol1     5Gi        RWX            vol-kubecliente   80m

NAME                                      READY   STATUS    RESTARTS   AGE
pod/despliegue-wp-5bdd879496-xntw2        1/1     Running   0          6m23s
pod/mariadb-deployment-66dc948d55-7rf6w   1/1     Running   0          7m11s
~~~

Se crea un post de prueba en wordpress:
![imga](https://github.com/PalomaR88/Gestion_de_usuarios_y_reglas_de_acceso_en_Kubernetes/blob/master/image/imga.png?raw=true)


A continuación, se va a borrar el pod de maria. Se va a realizar a través del administrador ya que el usuario Kubecliente solo puede borrarlos a través de despliegues:
~~~
debian@kubemaster:~$ kubectl delete pod mariadb-deployment-66dc948d55-t2jrm -n kubecliente
pod "mariadb-deployment-66dc948d55-t2jrm" deleted
~~~

Automáticamente se ha creado un nuevo pod:
~~~
debian@kubecliente:~/desp-wp$ kubectl get pods -n kubecliente
NAME                                  READY   STATUS    RESTARTS   AGE
despliegue-wp-5bdd879496-2mm5m        1/1     Running   0          12m
mariadb-deployment-66dc948d55-jjcd4   1/1     Running   0          17s
~~~

Y se comprueba que el post de prueba que se ha creado sigue disponible en la página de wordpress.



# 4. Herramientas auxiliares
Se ha investigado varias herramientas auxiliares que facilite el manejo de usuarios y permisos en clusters de Kubernetes.

## 4.1. Elastickube
Esta herramienta se encuentra disponible en el [repositorio oficial](https://github.com/ElasticBox/elastickube.git). Tras la instalación se ha llegado a la siguiente conclusión: la herramienta está desactualizada y no es compatible con las nuevas versiones de Kubernetes. Además, en la instalación se han producido diferentes cambios en la configuración de kubernetes que hacen inservible el cluster. 


## 4.2. Klum
Esta herramienta también se encuentra en su corresponddiente [repositorio oficial](https://github.com/ibuildthecloud/klum.git). Tras un primer acercamiento a esta herramienta se concluye que es muy reciente, lo que ocasiona que todavía no tiene ninguna utilidad.

## 4.3. Rakkess
Rakkes es un proyecto que ayuda a conocer las autorizaciones que poseen los usuarios. El proyecto se ha descargado desde el [repositorio oficial del proyecto](https://github.com/corneliusweig/rakkess) donde se explican las diversas formas de instalación que ofrece. 

Esta herramienta es muy intiutiva. Tras el binario **rakkess** se pueden utilizar las siguientes opciones:

|Opciones|Descripción
|--------|---------------
| `--verbs`| para indicar la acción o acciones.
| `--namespace` / `-n`| para restringir la lista a un espacio de nombres.
| `--verbosity`| establece el nivel de registro, por defecto es "warning". Se puede indicar "debug", "info", "warn", "error", "fatal", "panic".
| `--sa` / `--as`| para establecer la cuenta de servicio. 


En el siguiente ejemplo se recopila información sobre los permisos **CREATE** del usuario kubecliente en su namespace. 
~~~
debian@kubemaster:~$ rakkess -n kubecliente --as kubecliente --verbs create
NAME                                            CREATE
bindings                                        ✖
configmaps                                      ✖
controllerrevisions.apps                        ✖
cronjobs.batch                                  ✖
daemonsets.apps                                 ✖
deployments.apps                                ✔
endpoints                                       ✖
endpointslices.discovery.k8s.io                 ✖
events                                          ✖
events.events.k8s.io                            ✖
horizontalpodautoscalers.autoscaling            ✖
ingresses.extensions                            ✖
ingresses.networking.k8s.io                     ✖
jobs.batch                                      ✖
leases.coordination.k8s.io                      ✖
limitranges                                     ✖
localsubjectaccessreviews.authorization.k8s.io  ✖
networkpolicies.networking.k8s.io               ✖
persistentvolumeclaims                          ✔
poddisruptionbudgets.policy                     ✖
pods                                            ✖
podtemplates                                    ✖
replicasets.apps                                ✖
replicationcontrollers                          ✖
resourcequotas                                  ✖
rolebindings.rbac.authorization.k8s.io          ✖
roles.rbac.authorization.k8s.io                 ✖
secrets                                         ✔
serviceaccounts                                 ✖
services                                        ✔
statefulsets.apps                               ✖
~~~

En el siguiente ejemplo se lista los permisos de usuarios, grupos y cuentas de servicios sobre el recurso volúmenes persistentes. 
~~~
debian@kubemaster:~$ rakkess resource pv --namespace kubecliente
NAME                            KIND            SA-NAMESPACE  LIST  CREATE  UPDATE  DELETE
attachdetach-controller         ServiceAccount  kube-system   ✔     ✖       ✖       ✖
expand-controller               ServiceAccount  kube-system   ✔     ✖       ✔       ✖
generic-garbage-collector       ServiceAccount  kube-system   ✔     ✖       ✔       ✔
horizontal-pod-autoscaler       ServiceAccount  kube-system   ✔     ✖       ✖       ✖
kubecliente                     User                          ✔     ✖       ✔       ✖
namespace-controller            ServiceAccount  kube-system   ✔     ✖       ✖       ✔
persistent-volume-binder        ServiceAccount  kube-system   ✔     ✔       ✔       ✔
pv-protection-controller        ServiceAccount  kube-system   ✔     ✖       ✔       ✖
resourcequota-controller        ServiceAccount  kube-system   ✔     ✖       ✖       ✖
system:kube-controller-manager  User                          ✔     ✖       ✖       ✖
system:kube-scheduler           User                          ✔     ✖       ✔       ✖
system:masters                  Group                         ✔     ✔       ✔       ✔
~~~

## 4.4. Kubect-who-can
Este proyecto, cuya documento se encuentra en su [repositorio oficial](https://github.com/aquasecurity/kubectl-who-can), permite saber quiénes son los usuarios que pueden realizar una acción. Para su instalación se ha utilizado [krew](https://krew.sigs.k8s.io/docs/user-guide/quickstart/), una herramienta que permite instalar complementos de kubectl. 

El siguiente ejemplo responde a la pregunta, ¿quién puede crear pods en el namespace kubecliente?

~~~
debian@kubemaster:/tmp/tmp.t6yxdruWnR$ kubectl who-can create pods -n kubecliente
+ kubectl who-can create pods -n kubecliente
No subjects found with permissions to create pods assigned through RoleBindings

CLUSTERROLEBINDING                          SUBJECT                   TYPE            SA-NAMESPACE
cluster-admin                               system:masters            Group           
system:controller:daemon-set-controller     daemon-set-controller     ServiceAccount  kube-system
system:controller:job-controller            job-controller            ServiceAccount  kube-system
system:controller:persistent-volume-binder  persistent-volume-binder  ServiceAccount  kube-system
system:controller:replicaset-controller     replicaset-controller     ServiceAccount  kube-system
system:controller:replication-controller    replication-controller    ServiceAccount  kube-system
system:controller:statefulset-controller    statefulset-controller    ServiceAccount  kube-system
~~~

## 4.5. rbac-lookup
Este proyecto lista la relación de RBAC. De nuevo se ha utilizado krew para la instalación. La documentación se encuentra en el [repositorio oficial de Github](https://github.com/FairwindsOps/rbac-lookup).

Es tan sencillo como usar el comando seguido del usuario:
~~~
debian@kubemaster:/tmp/tmp.t6yxdruWnR$ kubectl rbac-lookup kubecliente
+ kubectl rbac-lookup kubecliente
SUBJECT        SCOPE          ROLE
kubecliente    kubecliente    Role/all-secrets-kubecliente
kubecliente    kubecliente    Role/all-PVClaim-kubecliente
kubecliente    kubecliente    Role/crear-serv-kubecliente
kubecliente    kubecliente    Role/despliegue-kubecliente
kubecliente    kubecliente    Role/get-serv-kubecliente
kubecliente    cluster-wide   ClusterRole/almacenamiento-kubecliente
~~~

# 5. Siguientes pasos

Este proyecto abarca los pilares fundamentales para comprender la administración y gestión de usuarios en Kubernetes pero se podría profundizar más en algunos aspectos que aquí se han explicado como las quotas o el establecimiento de permisos a través de grupos de usuarios. 

Una práctica muy interesante sería la realización de un script para crear usuarios de forma automática, incluso la creación automatizada de usuarios con un perfiles definidos. También, se podría indagar sobre las diferencias en la administración de usuarios y permisos en proveedores como Google, Amazon, Azure, etc. u otras plataformas como OpenShift. 


# 6. Conclusiones

Con la información recopilada en este documento se puede concluir que la característica principal de la administración de usuarios y permisos en Kubernetes es la versatilidad que ofrece. Pudiendo desgranar las opciones según las necesidades. Siendo, además, muy sencillo de estandarizar. 

Como se dijo en la introducción, este es un trabajo fundamental para un administrador de sistemas, pero se ha observado que no sólo es necesario el conocimiento sobre usuarios y permisos, sino que para realizar una correccta gestión se debe tener unos conocimientos mínimos sobre el funcionamiento de la API y los objetos del sistema. 



# 7. Webgrafía
> Para la elaboración de la webfragía se ha seguido el formato del estilo APA.

AGEE, Hannah. (s.f.). *User is unable to list persistent volumes*. Docker success center. Recuperado el 2 de junio de 2020, de [https://success.docker.com/article/user-unable-to-list-persistent-volumes](https://success.docker.com/article/user-unable-to-list-persistent-volumes).

AQUA eSOLUTIONS. (2020). *kubectl-who-can*. GitHub. Recuperado el 7 de junio de 2020, de [https://github.com/aquasecurity/kubectl-who-can/blob/master/README.md](https://github.com/aquasecurity/kubectl-who-can/blob/master/README.md).

BITNAMI. (s.f.). *RBAC in Kubernetes*. Cloud Native Computing Foundation. Recuperado el 30 de mayo de 2020, de [https://www.cncf.io/wp-content/uploads/2018/07/RBAC-Online-Talk.pdf](https://www.cncf.io/wp-content/uploads/2018/07/RBAC-Online-Talk.pdf).

CENTURYLINK. (2016). *ElasticKube - The Kubernetes Management Platform*. GitHub. Recuperado el 12 de abril de 2020, de [https://github.com/ElasticBox/elastickube/blob/master/README.md](https://github.com/ElasticBox/elastickube/blob/master/README.md).

FAIRWINDS. (2020). *RBAC LOOKUP*. GitHub. Recuperado el 7 de junio de 2020, de [https://github.com/FairwindsOps/rbac-lookup/blob/master/README.md](https://github.com/FairwindsOps/rbac-lookup/blob/master/README.md).

IES GONZALO NAZARENO. (2020). *08-administracion_basica*. GitHub. Recuperado el 17 de abril de 2020, de [https://github.com/iesgn/talleres_asir_iesgn/blob/master/kubernetes/presentaciones/08-administracion_basica.pdf](https://github.com/iesgn/talleres_asir_iesgn/blob/master/kubernetes/presentaciones/08-administracion_basica.pdf).

IES GONZALO NAZARENO. (2020). *09-Instalacion_paso_a_paso*. GitHub. Recuperado el 17 de abril de 2020, de [https://github.com/iesgn/talleres_asir_iesgn/blob/master/kubernetes/presentaciones/09-Instalacion_paso_a_paso.pdf](https://github.com/iesgn/talleres_asir_iesgn/blob/master/kubernetes/presentaciones/09-Instalacion_paso_a_paso.pdf).

MUÑOZ, José Domingo. (2018). *Desplegando una aplicación en Kubernetes*. PLEDIN 3.0. Recuperado el 12 de abril de 2020, de [https://www.josedomingo.org/pledin/2018/05/desplegando-una-aplicacion-en-kubernetes/](https://www.josedomingo.org/pledin/2018/05/desplegando-una-aplicacion-en-kubernetes/).

MUÑOZ, José Domingo. (2018). *Instalación de kubernetes con kubeadm*. PLEDIN 3.0. Recuperado el 12 de abril de 2020, de [https://www.josedomingo.org/pledin/2018/05/instalacion-de-kubernetes-con-kubeadm/](https://www.josedomingo.org/pledin/2018/05/instalacion-de-kubernetes-con-kubeadm/).

MUÑOZ, José Domingo. (2018). *Recursos de Kubernetes: Services*. PLEDIN 3.0. Recuperado el 12 de abril de 2020, de [https://www.josedomingo.org/pledin/2018/11/recursos-de-kubernetes-services/](https://www.josedomingo.org/pledin/2018/11/recursos-de-kubernetes-services/).

MUÑOZ, José Domingo. (2019) *Almacenamiento en Kubernetes. PersistentVolumen. PersistentVolumenClaims.* PLEDIN 3.0. Recuperado el 20 de mayo de 2020, de [https://www.josedomingo.org/pledin/2019/03/almacenamiento-kubernetes/](https://www.josedomingo.org/pledin/2019/03/almacenamiento-kubernetes/).

MUÑOZ, José Domingo. (2019). *Kubernetes. Desplegando WordPress con MariaDB*. PLEDIN 3.0. Recuperado el 12 de abril de 2020, de [https://www.josedomingo.org/pledin/2019/03/kubernetes-wordpress/](https://www.josedomingo.org/pledin/2019/03/kubernetes-wordpress/).

ORTIZ CARVAJAL, Juan Antonio. (2016). *ElasticKube, contenedor para el manejo empresarial de Kubernetes*. Adictos al trabajo. Recuperado el 12 de abril de 2020, de [https://www.adictosaltrabajo.com/2016/05/27/elastickube-contenedor-para-el-manejo-empresarial-de-kubernetes/](https://www.adictosaltrabajo.com/2016/05/27/elastickube-contenedor-para-el-manejo-empresarial-de-kubernetes/).

RANCHER LABS, Inc. (2020). *Klum - Kubernetes Lazy User Manager*. GitHub. Recuperado el 12 de abril de 2020, de [https://github.com/ibuildthecloud/klum/blob/master/README.md](https://github.com/ibuildthecloud/klum/blob/master/README.md).

SALMERON, Javier. (2018). *Demystifying RBAC in Kubernetes*. Cloud Native Computing Foundation. Recuperado el 16 de mayo de 2020, de [https://www.cncf.io/blog/2018/08/01/demystifying-rbac-in-kubernetes/](https://www.cncf.io/blog/2018/08/01/demystifying-rbac-in-kubernetes/).

THE LINUX FOUNDATION. (2016). *Dynamic Provisioning and Storage Classes in Kubernetes*. Kubernetes. Recuperado el 5 de junio de 2020, de [https://kubernetes.io/blog/2016/10/dynamic-provisioning-and-storage-in-kubernetes/](https://kubernetes.io/blog/2016/10/dynamic-provisioning-and-storage-in-kubernetes/).

THE LINUX FOUNDATION. (2018). *Limit Storage Consumption*. Kubernetes. Recuperado el 4 de junio de 2020, de [https://kubernetes.io/docs/tasks/administer-cluster/limit-storage-consumption/](https://kubernetes.io/docs/tasks/administer-cluster/limit-storage-consumption/).

THE LINUX FOUNDATION. (2019). *Access control*. Kubernetes. Recuperado el 30 de mayo de 2020, de [https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#access-control](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#access-control).

THE LINUX FOUNDATION. (2019). *Configure a Pod to Use a PersistentVolume for Storage*. Kubernetes. Recuperado el 30 de mayo de 2020, de [https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/).

THE LINUX FOUNDATION. (2019). *Espacios de nombres*. Kubernetes. Recuperado el 5 de mayo de 2020, de [https://kubernetes.io/es/docs/concepts/overview/working-with-objects/namespaces/](https://kubernetes.io/es/docs/concepts/overview/working-with-objects/namespaces/).

THE LINUX FOUNDATION. (2019). *Install and Set Up kubectl*. Kubernetes. Recuperado el 12 de abril de 2020, de [https://kubernetes.io/es/docs/tasks/tools/install-kubectl/#siguientes-pasos](https://kubernetes.io/es/docs/tasks/tools/install-kubectl/#siguientes-pasos).

THE LINUX FOUNDATION. (2020). *API OVERVIEW*. Kubernetes. Recuperado el 5 de junio de 2020, de [https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#storageclass-v1-storage](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#storageclass-v1-storage).


THE LINUX FOUNDATION. (2020). *Authenticating*. Kubernetes. Recuperado el 12 de abril de 2020, de [https://kubernetes.io/docs/reference/access-authn-authz/authentication/](https://kubernetes.io/docs/reference/access-authn-authz/authentication/).

THE LINUX FOUNDATION. (2020). *Certificates*. Kubernetes. Recuperado el 12 de abril de 2020, de [https://kubernetes.io/docs/concepts/cluster-administration/certificates/](https://kubernetes.io/docs/concepts/cluster-administration/certificates/)

THE LINUX FOUNDATION. (2020). *Configurar un Pod para Usar un Volume como Almacenamiento*. Kubernetes. Recuperado el 20 de mayo de 2020, de [https://kubernetes.io/es/docs/tasks/configure-pod-container/configure-volume-storage/](https://kubernetes.io/es/docs/tasks/configure-pod-container/configure-volume-storage/)

THE LINUX FOUNDATION. (2020). *Configure Quotas for API Objects*. Kubernetes. Recuperado el 6 de junio de 2020, de [https://kubernetes.io/docs/tasks/administer-cluster/quota-api-object/](https://kubernetes.io/docs/tasks/administer-cluster/quota-api-object/).

THE LINUX FOUNDATION. (2020). *Jobs - Run to Completion*. Kubernetes. Recuperado el 28 de abril de 2020, de [https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/)

THE LINUX FOUNDATION. (2020). *Krew*. GitHub. Recuperado el 2 de junio de 2020, de [https://github.com/kubernetes-sigs/krew/blob/master/README.md](https://github.com/kubernetes-sigs/krew/blob/master/README.md).

THE LINUX FOUNDATION. (2020). *Persistent Volumes*. Kubernetes. Recuperado el 30 de mayo de 2020, de [https://kubernetes.io/docs/concepts/storage/persistent-volumes/](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).

THE LINUX FOUNDATION. (2020). *Resource Quotas*. Kubernetes. Recuperado el 2 de junio de 2020, de [https://kubernetes.io/docs/concepts/policy/resource-quotas/](https://kubernetes.io/docs/concepts/policy/resource-quotas/).

THE LINUX FOUNDATION. (2020) *Role-based Access Control*. Helm. Recuperado el 16 de mayo de 2020, de [https://helm.sh/docs/topics/rbac/](https://helm.sh/docs/topics/rbac/).

THE LINUX FOUNDATION. (2020). *Storage Classes*. Kubernetes. Recuperado el 2 de junio de 2020, de [https://kubernetes.io/docs/concepts/storage/storage-classes/](https://kubernetes.io/docs/concepts/storage/storage-classes/).

THE LINUX FOUNDATION. (2020). *Using Admission Controllers*. Kubernetes. Recuperado el 5 de junio de 2020, de [https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass).

THE LINUX FOUNDATION. (2020) *Using RBAC Authorization*. Kubernetes. Recuperado el 13 de mayo de 2020, de [https://kubernetes.io/docs/reference/access-authn-authz/rbac/](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).

THE LINUX FOUNDATION. (2020). *Volumes*. Kubernetes. Recuperado el 2 de junio de 2020, de [https://kubernetes.io/docs/concepts/storage/volumes/](https://kubernetes.io/docs/concepts/storage/volumes/).

WEIG, Cornelius. (2020). *rakkess*. GitHub. Recuperado el 7 de junio de 2020, de [https://github.com/corneliusweig/rakkess/blob/master/README.md](https://github.com/corneliusweig/rakkess/blob/master/README.md).
