# 1. Introducción

## 1.1. Creación del escenario
Para los casos prácticos se ha utilizado un cluster de Kubernetes instalados sobre tres máquinas con sistemas operativos Debian Buster y una máquina cliente sobre el mismo sistema. Las caracterísitca son las siguientes:
- **kubemaster**: master del cluster de Kubernetes. 4GB de Ram de 40GB de disco.
- **kubeminion1** y **kubeminion2**: minions del cluster de Kubernetes. 2GB de RAM y 20GB de disco.
- **kubecliente**: cliente del cluster de kubernetes. 1GB de RAM y 20GB de disco.

Para la isntalación del cluster se ha seguido una [guía propia de instalación](https://github.com/PalomaR88/kubernete/blob/master/Practica.md#instalaci%C3%B3n-de-kubernetes).

El puerto de acceso a la API en un cluster de Kubernetes típico es en el 6443. Por lo tanto, se abrirán estos puertos en las diferentes máquinas. Otros puertos que se van a usar son el 80 para acceder a los servicios, 443 para los servicios a través de HTTPS y del 30000 al 40000 para aplicaciones con el servicio NodePort.

# 2. Control de acceso a la API
Para que los usuarios puedan ser autorizados para el acceso a la API se pasa por 3 etapas: autenticación, autorización y control de admisión.

Los clúster de Kubernetes tienen un certificado, que seuele estar autorfimado, y se incluyen en **$USER/.kube/config**. Este cerficado se escribe automáticamente cuando se crea un clúster y se puede compartir con otros usuarios.



## 2.1. Autenticación
Hay dos categorías de usuarios en Kubernetes: cuentas administradas por Kubernetes y usuarios normales administrados por un servicio externo e independiente como Keystone, Coocle Accounts o una lista de ficheros.

Aquí se trabajará con las cuentas administradas por Kubernetes. Se crean automáticamente por el servidor API o manualmente a través de una llamada. Éstas están vinculadas a un conjunto de credenciales almacenadas como Secrets.

Para autenticar las solicitudes de API Kubernetes utiliza certificados de cliente, tokens, proxy de autenticación o autenticación básica HTTP. Para las solicitud se utilizan los siguientes atributos: nombre de usuario, UID, grupos y campos adicionales.

Se pueden habilitar varios métodos de autenticación a la vez, pero, por lo general, deben usarse el meodo a través de tokens y otro. 

### 2.1.1. Certificados de cliente X509
Se debe generar certificados desde la línea de comandos a través de **easyrsa**, **openssl** o **cfssl**. En esta tarea se explica la creación de certificados a través de **openssl**. 

Estos certificados deben ser firmados por el certificado y la clave que proporciona Kubernetes. En el caso de tener instalado kubeadm se encuentra en /etc/kubernetes/pki/ y en minikube en ~/.minikube/. También se pueden crear credenciales propias, para ello se necesita una entidad certificadora. 

**Creación del certificado del cliente**
En primer lugar, el usuario tiene que genrar la clave y la petición de firma.
1. Generación de la clave
~~~
openssl genrsa -out <nombre_key>.key 2048
~~~

2. Generación de la petición de firma
~~~
openssl req -new -key <nombre_key>.key -out <nombre_csr>.csr -subj "/CN=<commonName_del_usuario>/O=<grupo>"
~~~

A continuación, un usuario con permisos de administrador del clúster firma y crea el certificado.
3. Creación del certificado
~~~
openssl x509 -req -in <nombre_csr>.csr -CA /<ruta_certificado_CA>/ca.crt -CAkey /<ruta_key_CA>/ca.key -CAcreateserial -out <nombre_certificado>.crt [-days <nº_días>]
~~~

4. Creación de los usuarios en el clúster (desde el administrador del clúster)
~~~
kubectl config set-credentials <usuario> --client-certificate=<nombre_certificado>.crt --client-key=<nombre_clave>.key
~~~

Para comprobar los usuarios que se han creado:
~~~
kubectl config view
~~~

#### 2.1.1.1. Caso práctico
Se instala kubectl en el nodo cliente y desde el master se dan permiso de lectura al fichero **/etc/kubernetes/admin.conf**:
~~~
debian@kubemaster:~$ sudo chmod 644 /etc/kubernetes/admin.conf
~~~

Desde el nodo cliente se va copiar el el fichero de configuración del master:
~~~
export IP_MASTER=172.22.200.133
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

Y se crea el directorio .kube para alojar el fichero de configuración:
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

En estos momentos, el nodo cliente funciona como el administrador porque se ha copiado el fichero de configuración sin configurar adecuadamente. El siguiente paso es la creación del usuario y la configuración: 

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


### 2.1.2. Tokens
#### 2.1.2.1. Caso práctico

## 2.2. Autorización
Creación de usuario en el clúster con autenticación de certificados:
~~~
kubectl config set-credentials <nombre_usuario> --client-certificate=<nombre_certificado>.crt --client-key=<nombre_clave>.key
~~~

Para ejecutar comando con un usuario concreto se utilizan los contextos. Para veer los contextos existentes se utiliza la siguinete orden:
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

Y para ver el contexto que se está usuando:
~~~
kubectl config current-context
~~~

### 2.2.1. Caso práctico
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


## 2.3. Autorización RBAC
El control de acceso basado en roles (RBAC) es un método para regular el acceso a los recursos en función de los roles de los usuarios. Para ello utiliza **rbac.authorization.k8s.io** que permite configurar dinámicamente políticas a través de la API de Kubernetes. 

### 2.3.1. Objetos API
La API RBAC declara cuatro tipos de objetos: Role, ClusterRole, RoleBinding y ClusterRoleBinding. Para trabajar con estos objetos, como todos los objetos de Kubernetes, se debe usar la API de Kubernetes. 

Los objetos de Kubernetes son entidades persistentes en el sistema de Kubernetes que se usan para representar el estado del cluster. Pueden describir:
- Qué palicaciones en contenedores se están ejecutando y en qué nodos.
- Los recursos disponibles para esas aplicaciones. 
- Las políticas sobre cómo se comportan esas aplicaciones (renicio, actualizaciones, tolerancia a fallos, etc.).

#### 2.3.1.1. Role y ClusterRole
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

Las dos claves para entender la sección **rules** son los recursos y las acciones, es decir, **Resources** y **Verbs**

Los recursos, **resources**, como se ha indicado anteriormente, son los conjunto de objetos API de Kubernetes disponibles en el cluster. A los que se hacen referencia se representan y accede a ellos mediante una cadena de nombre. Estos son algunos de los recursos: "pods", "secrets", "deployments", "services", "configmaps", "endpoints", "crontrabs", "jobs", "nodes", "replicasets", "configMaps", "autoScaler". Para indicar subrecursos se emplea la barra inclinada, por ejemplo, el subrecurso log de los pods se indica de la siguiente forma: "pods/log".

Con la opción **resourceNames** se puede consultar los recursos por nombre. Esta opción no se puede usar con las restricciones create o deletecollection por razones lógicas.

El conjunto de operaciones que se pueden ejecutar sobre los recursos se indican en **verbs**. Son, por ejemplo, "get", "watch", "create", "delete", "list", "patch", "update", "".

A continuación, se van a desarrollar algunos ejemplos de la sección rules para poner en práctica lo anteriormente explicado:

- Permitir que se puedan listar los pods, es decir, ejecutar el comando get pods:
~~~
rules:
 - apiGroups: ["*"]
   resources: ["pods"]
   verbs: ["list"]
~~~

- Permitir acceso de solo lectura en los pods, sin poder eliminar pods directamente pero sí a través de implementaciones,**deployments**:
~~~
 rules:
 - apiGroups: ["*"]
   resources: ["pods"]
   verbs: ["list","get","watch"]
 - apiGroups: ["extensions","apps"]
   resources: ["deployments"]
   verbs: ["get","list","watch","create","update","patch","delete"]
~~~

****************************


#### 2.3.1.2. RoleBinding y ClusterRoleBinding
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


### 2.3.2. Caso práctico
Para este caso práctico vamos a continuar con el contexto y usuario que se ha creado anteriormente: kubecliente. Con este usuario se va a desplegar una aplicación Wordpress, con una base de datos, sobre volúmenes persistentes. 

Con este usuario no se puede hacer nada, ya que no tiene permisos para nada, ni en su propio namespace:
~~~
debian@kubecliente:~$ kubectl get pods -n kubecliente
Error from server (Forbidden): pods is forbidden: User "kubecliente" cannot list resource "pods" in API group "" in the namespace "kubecliente"
~~~

Se van a establecer unas normas para que el usuario pueda utilizar este espacio de nombres, es decir, se va a crar un rol a través de un fichero de configuración que va a permitir, en el namespace correspondiente, unas funciones básicas, que son: ver y listar pods y ver, listar, crear, actualizar, borrar, etc. despliegues.
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

Tras a creación sel rol se asigna a un usuario, para ello se crea un objeto RoleBinding, configurado en un fichero con extensión yaml con la siguiente sintaxis:
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

El siguiente paso será crear un servicio tipo ClusterIP, que posteriormente conectará el pod encargado de la base de datos de MariaDB con el pod que alojará Wordpress.

Para ello se crea un fichero .yaml con el siguiente contenido:
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

Y el correspondiente fichero, para aplicar la regla al usuario, es:
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

Lo siguiente será crear un fichero donde se indican los datos para MariaDB
 que se guardarán en un secreto:
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

El siguiente fichero crea un servicio NodePort, es decir, que mapea el puerto 80 del contenedor a un puerto entre el 30000 y el 40000. Como se va a crear un servicio y este usuario tiene permisos para ello, Kubernetes permite realizar esta acción :
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

Este despliegue utiliza dos volúmenes persistentes que han sido creados con anterioridad por kubemaster. Para la creación de estos volúmenes se ha seguido una [guía propia](https://github.com/PalomaR88/Volumenes-persistentes-kubernetes/blob/master/Practica.md)
*
*
*
*
*
*
***************sigo por aquí, que es en el otro repositorio, creando los volumenes persistentes

***************************************

PASOS PARA CREAR VOLUMEN PERSITENTE PROBAR PRIMERO EN EL MASTER

--------------------------


-------------------------------------------------

Creación de volḿenes persistentes:
~~~
apiVersion: v1
kind: PersistentVolume
metadata:
  name: volumen5
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /home/shared/volumen5
    server: 10.0.0.3
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: volumen6
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /home/shared/volumen6
    server: 10.0.0.3
~~~

Crear la llamada wordpress:
~~~
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
  namespace: prueba-wp
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
~~~

Crear llamada maria:
~~~
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb-pvc
  namespace: prueba-wp
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
~~~


Deployment de mariadb definiendo el volumen:
~~~
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb-deployment
  namespace: prueba-wp
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
            - name: volumen5
              mountPath: /var/lib/mysql
      volumes:
        - name: volumen5
          persistentVolumeClaim:
            claimName: mariadb-pvc
~~~

Creación del deployment de wordpress definiendo el volumen:
~~~
apiVersion: apps/v1
kind: Deployment
metadata:
  name: despliegue-wp
  namespace: prueba-wp
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
            - name: volumen6
              mountPath: /var/www/html
      volumes:
        - name: volumen6
          persistentVolumeClaim:
            claimName: wordpress-pvc
~~~


Ver cositas:
~~~
debian@kubemaster:~/prueba-wp$ kubectl get deployment,service,pv,pvc,pods -n prueba-wp
NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/despliegue-wp        1/1     1            1           10s
deployment.apps/mariadb-deployment   1/1     1            1           51s

NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/mariadb-service     ClusterIP   10.107.250.109   <none>        3306/TCP                     11m
service/wordpress-service   NodePort    10.102.234.54    <none>        80:31501/TCP,443:32363/TCP   9m3s

NAME                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS        CLAIM                     STORAGECLASS   REASON   AGE
persistentvolume/volumen1   1Gi        RWX            Recycle          Terminating   prueba/nfs-pvc                                    4d21h
persistentvolume/volumen2   5Gi        RWX            Recycle          Bound         prueba/wordpress-pvc                              2d22h
persistentvolume/volumen3   5Gi        RWX            Recycle          Bound         wordpress/wordpress-pvc                           21m
persistentvolume/volumen4   5Gi        RWX            Recycle          Bound         prueba-wp/wordpress-pvc                           21m
persistentvolume/volumen5   5Gi        RWX            Recycle          Bound         prueba-wp/mariadb-pvc                             4m45s
persistentvolume/volumen6   5Gi        RWX            Recycle          Available                                                       4m45s

NAME                                  STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/mariadb-pvc     Bound    volumen5   5Gi        RWX                           103s
persistentvolumeclaim/wordpress-pvc   Bound    volumen4   5Gi        RWX                           2m6s

NAME                                      READY   STATUS    RESTARTS   AGE
pod/despliegue-wp-65d9cb48c4-bdw4q        1/1     Running   0          9s
pod/mariadb-deployment-5fd66dbcc9-5pwwx   1/1     Running   0          51s
~~~



Probar:
~~~
kubectl get pods -n prueba-wp
kubectl delete pod -n wordpress mariadb-deployment-59f59b...


Fichero pv.yaml para crear el volumen persistente:
~~~
apiVersion: v1
kind: PersistentVolume
metadata:
  name: volumen1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /home/shared/vol1
    server: 10.0.0.3
~~~

~~~
debian@kubemaster:~$ kubectl create -f pv.yaml
persistentvolume/volumen1 created
~~~
**************************************
*
*
*
*
*
*
*



**********************************************
Se va a crear un espacio de nombre para que pueda trabajar el usuario.
~~~
kubepaloma@kubeprueba:~$ kubectl create namespace kubepaloma
namespace/kubepaloma created
~~~

Y se crea una RBAC a través de un fichero de configuración, en este caso se llama RBAC_kubepaloma.yaml con el siguiente contenido:
~~~
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
    name: kubepaloma
    namespace: kubepaloma
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["jobs"]
  verbs: ["watch", "create", "update", "patch", "delete"]
~~~

> La configuración del fichero no parece tener mucho sentido, pero es para ejemplificar.

En este caso se le va a otorgar permisos de realizar las acciones de get, list y watch los pods y de watch, create, update, patch y delete sobre los jobs.

A continuación, se acplica el rol:
~~~
kubepaloma@kubeprueba:~$ kubectl apply -f RBAC_kubepaloma.yaml 
role.rbac.authorization.k8s.io/kubepaloma created
~~~

Y se comprueba la creación del rol:
~~~
kubepaloma@kubeprueba:~$ kubectl get role -n kubepaloma
NAME         CREATED AT
kubepaloma   2020-04-22T15:46:25Z
~~~

Por último, hay que asignar el rol al usuario. Se va a realizar a través de un fichero yaml que llamaremos kubepaloma.yaml:
~~~
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: full-control
  namespace: kubepaloma
subjects:
- kind: User
  name: kubepaloma
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: kubepaloma
  apiGroup: rbac.authorization.k8s.io
~~~

Se aplica:
~~~
kubepaloma@kubeprueba:~$ kubectl apply -f kubepaloma.yaml
rolebinding.rbac.authorization.k8s.io/full-control unchanged
~~~

Se va a comprobar si se ha realizado bien la RBAC. En primer lugar, se cambia de contexto:
~~~
kubepaloma@kubeprueba:~$ kubectl config use-context kubepaloma
Switched to context "kubepaloma".
~~~

Vamos a listar por pods, algo que está permitido en el namespace kubepaloma:
~~~
kubepaloma@kubeprueba:~$ kubectl get pods
Error from server (Forbidden): pods is forbidden: User "kubepaloma" cannot list resource "pods" in API group "" in the namespace "default"
~~~

No nos permita listar los pods, puesto que solo se tiene permiso en el namespace kubepaloma:
~~~
kubepaloma@kubeprueba:~$ kubectl get pods -n kubepaloma
No resources found in kubepaloma namespace.
~~~

Pero no nos deja listar los jobs:
~~~
kubepaloma@kubeprueba:~$ kubectl get jobs -n kubepaloma
Error from server (Forbidden): jobs.batch is forbidden: User "kubepaloma" cannot list resource "jobs" in API group "batch" in the namespace "kubepaloma"
~~~

# 3. Herramientas auxiliares
Se ha investigado sobre dos herramientas auxiliares que facilite el manejo de usuarios y permisos en clusters de Kubernetes que son Elastickube y Klum.

# 3.1. Elastickube
Esta herramienta se encuentra disponible en el [repositorio oficial](https://github.com/ElasticBox/elastickube.git). Tras la instalación se ha llegado a la siguiente conclusión: la herramienta está desactualizada y no es compatible con las nuevas versiones de Kubernetes. Además, en la instalación se han producido diferentes cambios en la configuración de kubernetes que hacen inservible el cluster. 


# 3.2. Klum
Esta herramienta también se encuentra en su corresponddiente [repositorio oficial](https://github.com/ibuildthecloud/klum.git). Tras un primer acercamiento a esta herramienta se concluye que es muy nueva??(no me gusta este adjetivo, lo tengo que cambiar) lo que ocasiona que todavía no tiene ninguna utilizadad.


https://kubernetes.io/docs/reference/access-authn-authz/rbac/
