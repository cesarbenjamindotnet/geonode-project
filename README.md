# {{ project_name|title }}

GeoNode template project. Generates a django project with GeoNode support.

# Manual de instalación

## Requisitos

Servidor físico o máquina virtual con 8 CPUs y 16 GB de RAM y 500GB de espacio en disco. El servidor debe tener el sistema operativo Ubuntu Server LTS vigente (al momento de este documento, es la versión 22.04.3 LTS).

# Máquina Virtual

Para correr el servicio es necesario utilizar una máquina virtual en GCP de tipo N2 con 8 CPUs y 14 GB de RAM en la zona us-central1-c, con una interfaz de red de tipo gVNIC. 

El proyecto donde vive la máquina virtual debe correr en un proyecto vinculado a la red compartida en la vpc de producción y debe tener reservada una IP pública externa

Implementación actual 

La implementación actual corre dentro del proyecto mty-geonode para producción.

El proyecto mty-geonode está vinculado a la VPC compartida del proyecto mty-network-trv en la red de VPC denominada vpc-prod (10.200.0.0/18).

Respecto a la red se tienen reservadas una ip externa y una interna, la ip interna es 10.200.0.5 y la externa es 35.226.137.213. Además se utilizan las etiquetas de red allow-iap-ssh para poder conectar al servidor por SSH y allow-web, para permitir acceso desde el exterior a los puertos TCP de HTTP.

# Cloud Storage

## Bucket

Para almacenar archivos de manera persistente, se utilizan Buckets de Cloud Storage, para este caso se creó un bucket llamado **mty-geonode-volumes** multiregión con Autoclass con la opción “Habilitar la opción para permitir las transiciones de objetos a las clases Coldline y Archive” activada. Con prevención de acceso público a este bucket uniforme. Con prevención de acceso público. Con la misma configuración hay que crear un bucket llamado **mty-geonode-media**. 

```mty-geonode-media``` será utilizado para el almacenamiento de subida desde django  
```mty-geonode-volumes``` será usado para todo lo demás

Hay que montar el bucket volumes en fstab para poder montar los volúmenes de docker, para ello agregar al final del archivo /etc/fstab la siguiente línea: 

```
mty-geonode-volumes /mnt/mty-geonode-volumes gcsfuse rw,\_netdev,allow\_other,file\_mode=777,dir\_mode=777,uid=0,gid=0,key\_file=/root/gcs\_volumes\_sa\_keyfi  
le.json
```

Además, hay que crear algunos directorios dentro del bucket:
```
mkdir /mnt/mty-geonode-volumes/datosmty-backup-restore  
mkdir /mnt/mty-geonode-volumes/datosmty-dbbackups  
mkdir /mnt/mty-geonode-volumes/datosmty-geoserver-styles  
mkdir /mnt/mty-geonode-volumes/datosmty-geoserver-workspaces  
mkdir /mnt/mty-geonode-volumes/datosmty-statics  
mkdir /mnt/mty-geonode-volumes/datosmty-uploaded  
mkdir /mnt/mty-geonode-volumes/datosmty-backup-restore
```
## Service Account

Para acceder a los buckets hay que crear un archivo de cuenta de servicio que nombramos **gcs\_sa\_keyfile.json**, para lo cual se crea una cuenta de servicio en el apartado **IAM** de la consola de GCP con el nombre **gcs-objects-user** con el rol **Usuario de objetos de almacenamiento**. Una vez creada la cuenta de servicio hay que generar y descargar un archivo llave para dicha cuenta de servicio. renombrarlo y ubicarlo en la raíz del directorio **datosmty** que se creó al inicializar el proyecto con django-admin.

Hay que tener cuidado de no publicar el archivo de cuenta de servicio en el repositorio de código. este se debe colocar posteriormente de manera manual

## Docker

Docker es un sistema de manejo de contenedores, un contenedor es una forma de distribuir aplicaciones en la modalidad código como servicio. Es necesario tener instalado Docker en el servidor donde corre el sistema. Para instalar Docker hay que llevar a cabo los siguientes pasos en el servidor, habiéndose conectado por SSH.

## Actualizar el sistema

Siempre es recomendado actualizar cualquier sistema, así se encuentre recién instalado, también es recomendado aplicar las últimas actualizaciones periódicamente.

Ejecutar los siguientes comandos, uno a la vez, cuando finalice uno, introducir el siguiente, hasta el final.
```
- **sudo apt-get update**  
- **sudo apt-get upgrade \-y**
```
  *en caso de que se nos pregunte por servicios a reiniciar dejar todo como está y simplemente dar \[ok\]*
```
- **sudo apt autoremove \-y**
```
## Eliminar cualquier rastro de Docker

Considerando un escenario donde no se trate de un servidor recién instalado y que pudiera tener docker instalado en una versión antigua, podemos actualizar a la versión más reciente, pero primero hay que eliminar todos los posibles remanentes que pudieran existir instalados en el sistema operativo.

Ejecutar los siguientes comandos, uno a la vez, cuando finalice uno, introducir el siguiente, hasta el final.
```
- **sudo apt remove docker-desktop \--purge**  
- **sudo apt remove docker.io \--purge**  
- **sudo rm \-r $HOME/.docker/desktop**  
- **sudo rm /usr/local/bin/com.docker.cli**
```
  *Es posible que algunos de los comandos del punto anterior marquen algún error, esto es en caso de que los paquetes referidos no hayan sido encontrados, en caso de error no es un problema, solo nos está informando que no puede eliminar algo que no encontró, se puede ignorar los mensajes de error sin mayor problema.*

## Descargar repositorio de Docker

Ubuntu Server 22.04 tiene en su repositorio oficial docker, pero es una versión muy antigua (20.10); a continuación vamos a utilizar un método no oficial de Ubuntu pero si oficial de docker para instalar en a modo de Ubuntu, Docker, pero en la versión más actual disponible.

Ejecutar los siguientes comandos, uno a la vez, cuando finalice uno, introducir el siguiente, hasta el final.
```
- **sudo apt-get install ca-certificates curl gnupg lsb-release**  
- **sudo mkdir \-p /etc/apt/keyrings**  
- **curl \-fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg \--dearmor \-o /etc/apt/keyrings/docker.gpg**  
- **echo "deb \[arch=$(dpkg \--print-architecture) signed-by=/etc/apt/keyrings/docker.gpg\] https://download.docker.com/linux/ubuntu $(lsb\_release \-cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list \> /dev/null**  
- **sudo apt update**  
- **sudo apt install docker-ce \-y**
```

## Ejecutar Docker como usuario no root

Ejecutar los siguientes comandos, uno a la vez, cuando finalice uno, introducir el siguiente, hasta el final.
```
- **sudo groupadd docker**  
- **sudo usermod \-aG docker $USER**
```
## Instalar herramientas iniciales

Debemos instalar algunas herramientas, librerías y dependencias iniciales necesarias para construir el sistema e instalarlo, build-essential es un metapaquete que contiene varias herramientas de desarrollo y compilación; python3-pip es un manejador de paquetes para python con el cual vamos a instalar virtualenv; y virtualenv es una herramienta para crear entornos virtuales de python. Es importante crear entornos virtuales con python en lugar de utilizar el sistema nativo para no tocar configuraciones del sistema y de ese modo evitar conflictos si eventualmente otras cosas en el mismo servidor pudieran necesitar python con configuraciones muy particulares.

Ejecutar los siguientes comandos, uno a la vez, cuando finalice uno, introducir el siguiente, hasta el final.
```
- **sudo apt install build-essential python3-pip \-y**  
- **sudo pip install virtualenv**  
- **sudo reboot**
```
  *En este punto hay que reiniciar el servidor para activar algunas variables de entorno que no se habilitan de manera automática luego de instalar los paquetes previos.*

Instalar GCSFUSE
```
- **export GCSFUSE\_REPO=gcsfuse-\`lsb\_release \-c \-s\`**  
- **echo "deb \[signed-by=/usr/share/keyrings/cloud.google.asc\] https://packages.cloud.google.com/apt $GCSFUSE\_REPO main" | sudo tee /etc/apt/sources.list.d/gcsfuse.list**  
- **curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo tee /usr/share/keyrings/cloud.google.asc**  
- **sudo apt-get update**  
- **sudo apt-get install gcsfuse**   
```

## Crear directorios de trabajo

En estos directorios, en **venv** crearemos un entorno virtual de python para inicializar el proyecto, en **app** se va a inicializar el proyecto para hacer la instalación.

Ejecutar los siguientes comandos, uno a la vez, cuando finalice uno, introducir el siguiente, hasta el final. 
```
- **mkdir \~/src/geonode/venv \-p**  
- **mkdir \~/src/geonode/app \-p**
```
## Activar entorno virtual de python

Dentro del directorio **\~/src/geonode/venv** vamos a crear un entorno virtual de python denominado **DATOSMTY**, el nombre es completamente arbitrario pero se utiliza así debido al contexto que nos ocupa.
```
- **cd \~/src/geonode/venv**  
- **virtualenv DATOSMTY \-p /usr/bin/python3**  
- **source DATOSMTY/bin/activate**
```
## Preparar entorno para construir imagen del contenedor

Ahora que ya tenemos un entorno virtual activo, podemos descargar el código inicial para arrancar el proyecto para construir la imagen del contenedor

```
- **pip install Django==4.2.10**  
- **cd \~/src/geonode/app**  
- **mkdir datosmty**  
- **GN\_VERSION=datos\_mty**  
- **django-admin startproject \--template=https://github.com/gobiernodigitalmonterrey/geonode-project/archive/refs/heads/$GN\_VERSION.zip \-e py,sh,md,rst,json,yml,ini,env,sample,properties \-n monitoring-cron \-n Dockerfile datosmty datosmty**
```


## Recommended: Track your changes

Step 1. Install Git (for Linux, Mac or Windows).

Step 2. Init git locally and do the first commit:

```bash
git init
git add *
git commit -m "Initial Commit"
```

Step 3. Set up a free account on github or bitbucket and make a copy of the repo there.

## Hints: Configuring `requirements.txt`

You may want to configure your requirements.txt, if you are using additional or custom versions of python packages. For example

```python
Django==3.2.16
git+git://github.com/<your organization>/geonode.git@<your branch>
```

## Increasing PostgreSQL Max connections

In case you need to increase the PostgreSQL Max Connections , you can modify
the **POSTGRESQL_MAX_CONNECTIONS** variable in **.env** file as below:

```
POSTGRESQL_MAX_CONNECTIONS=200
```

In this case PostgreSQL will run accepting 200 maximum connections.

## Test project generation and docker-compose build Vagrant usage

Testing with [vagrant](https://www.vagrantup.com/docs) works like this:
What vagrant does:

Starts a vm for test on docker swarm:
    - configures a GeoNode project from template every time from your working directory (so you can develop directly on geonode-project).
    - exposes service on localhost port 8888
    - rebuilds everytime everything with cache [1] to avoid banning from docker hub with no login.
    - starts, reboots to check if docker services come up correctly after reboot.

```bash
vagrant plugin install vagrant-reload
#test things for docker-compose
vagrant up
# check services are up upon reboot
vagrant ssh geonode-compose -c 'docker ps'
```

Test geonode on [http://localhost:8888/](http://localhost:8888/)

To clean up things and delete the vagrant box:

```bash
vagrant destroy -f
```

## Test project generation and Docker swarm build on vagrant

What vagrant does:

Starts a vm for test on docker swarm:
    - configures a GeoNode project from template every time from your working directory (so you can develop directly on geonode-project).
    - exposes service on localhost port 8888
    - rebuilds everytime everything with cache [1] to avoid banning from docker hub with no login.
    - starts, reboots to check if docker services come up correctly after reboot.

To test on a docker swarm enable vagrant box:

```bash
vagrant up
VAGRANT_VAGRANTFILE=Vagrantfile.stack vagrant up
# check services are up upon reboot
VAGRANT_VAGRANTFILE=Vagrantfile.stack vagrant ssh geonode-compose -c 'docker service ls'
```

Test geonode on [http://localhost:8888/](http://localhost:8888/)
Again, to clean up things and delete the vagrant box:

```bash
VAGRANT_VAGRANTFILE=Vagrantfile.stack vagrant destroy -f
```

for direct deveolpment on geonode-project after first `vagrant up` to rebuild after changes to project, you can do `vagrant reload` like this:

```bash
vagrant up
```

What vagrant does (swarm or comnpose cases):

Starts a vm for test on plain docker service with docker-compose:
    - configures a GeoNode project from template every time from your working directory (so you can develop directly on geonode-project).
    - rebuilds everytime everything with cache [1] to avoid banning from docker hub with no login.
    - starts, reboots.

[1] to achieve `docker-compose build --no-cache` just destroy vagrant boxes `vagrant destroy -f`

