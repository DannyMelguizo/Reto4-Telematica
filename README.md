# **ST0263 Tópicos Especiales en Telemática**

# **Estudiante**: Daniel Melguizo Roldan, dmelguizor@eafit.edu.co

# **Profesor**: Juan Carlos Montoya Mendoza, jcmontoy@eafit.edu.co

*******

### **RETO 4**
Es un reto que consiste en el despligue de una aplicacion web desarrollada en wordpress, la aplicacion debera ser desplegada utilizando kubernetes y debera contar con lo siguiente, dos pods para instanciar la aplicacion de WordPress, un balanceador de carga que redirija las peticiones a dichos pod, un NFS en el cual se alojaran los recursos de la aplicacion de WordPress y una base de datos.

Para el desarrollo de este reto se utilizo AWS, usando los servicios que ofrece como Elastic Kubernetes Service (EKS) y Elastic File System (EFS). Ademas utilizaremos Clound9 para acceder al Cluster y realizar toda la configuracion.

Para la creacion del cluster se establecio la siguiente configuracion.
* Version de Kubernetes: **1.29**
* Rol de servicio de cluster: **LabRole**
* VPC: **Se utilizo la Default**
* Subredes: **Se utilizaron las Default**

Ingresaremos a Cloud9 y crearemos una maquina virtual, para evitar errores en la creacion es necesario permitir las conexiones a traves del Secure Shell (SSH)

En la consola de Clound9 ejecutaremos los siguientes comandos para instalar las dependencias necesarias.

```ssh
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/bin
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.16.8/2020-04-16/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
aws eks update-kubeconfig --name #nombre-de-tu-cluster --region us-east-1
```

En la consola de Cloud9 hacemos lo siguiente, vamos al engranaje y en el apartado de AWS Setttings, desactivar "AWS managed temporary credentials"

Copiamos las credenciales otorgadas por AWS Academy en AWS Details, copiamos todo el AWS CLI y lo pegamos en el Cloud9 dentro del archivo ~/.aws/credentials, en caso de que ocurra algun error, copiar el siguiente comando y agregar las credenciales manualmente

```ssh
aws configure
```
En caso tal volver a modificar el archivo ~/.aws/credentials ya que se desconfigura al ejecutar el comando aws configure.

Crearemos un EFS en la VPC por defecto que ofrece AWS. Adicionalmente crearemos un grupo de seguridad en el cual habilitaremos el puerto 2049 que corresponde a NFS, asegurarse de colocarlo en Inbound rules, por ultimo en el EFS creado agregaremos en el apartado de Network el grupo de seguridad creado a las diferentes zonas de disponibilidad habilitadas.

Ahora instalaremos los drivers necesarios para permitir la conexion al EFS con el siguiente comando

```ssh
kubectl apply -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.3"
```

En la parte del cluster, crearemos un grupo de nodos, en el IAM role colocaremos el LabRole, que las maquinas sean de Amazon Linux, la capacidad sera On-Demand y el tipo de instancias puede ser t2.micro o t2.medium, en mi caso utilice el t2.medium y lanzaremos un tamaño minimo de 5 maquinas.

Una vez este todo listo cambiaremos los siguientes archivos dentro de la carpeta YAML

configmap.yaml

```yaml
data:
  nginx.conf: |
    worker_processes auto;
    pid /run/nginx.pid;
    error_log /var/log/nginx/error.log;

    events {
        worker_connections 768;
    }

    http{

        server {
            listen 80 ;
            listen [::]:80 ;

            server_name tu-dominio.com www.tu-dominio.com;

            location / {
                proxy_pass http://wordpress-service;
            }
        }
    }
```

Cambiaremos el server_name por el dominio que tengamos

efs-pvc.yaml

```yaml
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  storageClassName: efs-sc
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: efs.csi.aws.com
    volumeHandle: #elastic-file-system-id
```

Cambiaremos el volumeHandle por el id de nuestro EFS

Iniciamos cada uno de los yaml dentro de la carpeta yaml y listo.

### **Referencias**
* https://www.youtube.com/@ProfeSantosCloud/featured