# 1. Introducción

## 1.1. Creación del escenario
Para los casos prácticos se ha utilizado un cluster de Kubernetes instalados sobre tres máquinas con sistemas operativos Debian Buster y una máquina cliente sobre el mismo sistema. Las caracterísitca son las siguientes:
- **kubemaster**: master del cluster de Kubernetes. 4GB de Ram de 40GB de disco.
- **kubeminion1** y **kubeminion2**: minions del cluster de Kubernetes. 2GB de RAM y 20GB de disco.
- **kubecliente**: cliente del cluster de kubernetes. 1GB de RAM y 20GB de disco.

Para la isntalación del cluster se ha seguido una [guía propia de instalación](https://github.com/PalomaR88/kubernete/blob/master/Practica.md#instalaci%C3%B3n-de-kubernetes).


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
debian@kubecliente:~$ kubectl config set-credentials kubecliente --client-certificate=kubecliente.crt --client-key=kubecliente.key
User "kubecliente" set.
~~~


Y se comprueba que el usuario se ha creado correctamente:
~~~
debian@kubecliente:~$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://172.22.200.133:6443
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
debian@kubecliente:~$ kubectl config set-context kubecliente --cluster=kubernetes --user=kucliente
Context "kubecliente" created.
~~~

Y se comprueba que se ha creado:
~~~
debian@kubecliente:~$ kubectl config get-contexts
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
          kubecliente                   kubernetes   kucliente          
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin   
~~~



## 2.3. Control de admisión
 ****************************************
 ** AQUÍ EXPLICAR BIEN Y PONER OPCIONES**
 ****************************************

### 2.3.1. Caso práctico
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


