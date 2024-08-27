# Geonode: mide

GeoNode template project. Generates a django project with GeoNode support.

# Manual de instalación

## Requisitos

Servidor físico o máquina virtual con 8 CPUs y 16 GB de RAM y 500GB de espacio en disco. El servidor debe tener el sistema operativo Ubuntu Server LTS vigente (al momento de este documento, es la versión 22.04.3 LTS).

# Máquina Virtual

Para correr el servicio es necesario utilizar una máquina virtual en GCP de tipo N2 con 8 CPUs y 14 GB de RAM en la zona us-central1-c, con una interfaz de red de tipo gVNIC. 

El proyecto donde vive la máquina virtual debe correr en un proyecto vinculado a la red compartida en la vpc de producción y debe tener reservada una IP pública externa

**Implementación actual**

La implementación actual corre dentro del proyecto mty-geonode para producción.

El proyecto mty-geonode está vinculado a la VPC compartida del proyecto mty-network-trv en la red de VPC denominada ```vpc-prod (10.200.0.0/18)```.

Respecto a la red se tienen reservadas una ip externa y una interna, la ip interna es  ```10.200.0.5``` y la externa es  ```35.226.137.213```. Además se utilizan las etiquetas de red  ```allow-iap-ssh``` para poder conectar al servidor por SSH y  ```allow-web```, para permitir acceso desde el exterior a los puertos TCP de HTTP.

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
- sudo apt-get update
- sudo apt-get upgrade \-y
```
  *en caso de que se nos pregunte por servicios a reiniciar dejar todo como está y simplemente dar \[ok\]*
```
- sudo apt autoremove \-y
```
## Eliminar cualquier rastro de Docker

Considerando un escenario donde no se trate de un servidor recién instalado y que pudiera tener docker instalado en una versión antigua, podemos actualizar a la versión más reciente, pero primero hay que eliminar todos los posibles remanentes que pudieran existir instalados en el sistema operativo.

Ejecutar los siguientes comandos, uno a la vez, cuando finalice uno, introducir el siguiente, hasta el final.

```
- sudo apt remove docker-desktop \--purge
- sudo apt remove docker.io \--purge 
- sudo rm \-r $HOME/.docker/desktop  
- sudo rm /usr/local/bin/com.docker.cli
```

  *Es posible que algunos de los comandos del punto anterior marquen algún error, esto es en caso de que los paquetes referidos no hayan sido encontrados, en caso de error no es un problema, solo nos está informando que no puede eliminar algo que no encontró, se puede ignorar los mensajes de error sin mayor problema.*

## Descargar repositorio de Docker
Ubuntu Server 22.04 tiene en su repositorio oficial docker, pero es una versión muy antigua (20.10); a continuación vamos a utilizar un método no oficial de Ubuntu pero si oficial de docker para instalar en a modo de Ubuntu, Docker, pero en la versión más actual disponible.

Ejecutar los siguientes comandos, uno a la vez, cuando finalice uno, introducir el siguiente, hasta el final.

```
- sudo apt-get install ca-certificates curl gnupg lsb-release
- sudo mkdir \-p /etc/apt/keyrings
- curl \-fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg \--dearmor \-o /etc/apt/keyrings/docker.gpg
- echo "deb \[arch=$(dpkg \--print-architecture) signed-by=/etc/apt/keyrings/docker.gpg\] https://download.docker.com/linux/ubuntu $(lsb\_release \-cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list \> /dev/null 
- sudo apt update  
- sudo apt install docker-ce \-y
```

## Ejecutar Docker como usuario no root
Ejecutar los siguientes comandos, uno a la vez, cuando finalice uno, introducir el siguiente, hasta el final.

```
- sudo groupadd docker
- sudo usermod \-aG docker $USER
```

## Instalar herramientas iniciales
Debemos instalar algunas herramientas, librerías y dependencias iniciales necesarias para construir el sistema e instalarlo, build-essential es un metapaquete que contiene varias herramientas de desarrollo y compilación; python3-pip es un manejador de paquetes para python con el cual vamos a instalar virtualenv; y virtualenv es una herramienta para crear entornos virtuales de python. Es importante crear entornos virtuales con python en lugar de utilizar el sistema nativo para no tocar configuraciones del sistema y de ese modo evitar conflictos si eventualmente otras cosas en el mismo servidor pudieran necesitar python con configuraciones muy particulares.

Ejecutar los siguientes comandos, uno a la vez, cuando finalice uno, introducir el siguiente, hasta el final.

```
- sudo apt install build-essential python3-pip \-y 
- sudo pip install virtualenv  
- sudo reboot
```
  *En este punto hay que reiniciar el servidor para activar algunas variables de entorno que no se habilitan de manera automática luego de instalar los paquetes previos.*

Instalar GCSFUSE
```
- export GCSFUSE\_REPO=gcsfuse-\`lsb\_release \-c \-s\` 
- echo "deb \[signed-by=/usr/share/keyrings/cloud.google.asc\] https://packages.cloud.google.com/apt $GCSFUSE\_REPO main" | sudo tee /etc/apt/sources.list.d/gcsfuse.list
- curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo tee /usr/share/keyrings/cloud.google.asc  
- sudo apt-get update
- sudo apt-get install gcsfuse 
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
- cd \~/src/geonode/venv  
- virtualenv DATOSMTY \-p /usr/bin/python3
- source DATOSMTY/bin/activate
```
## Preparar entorno para construir imagen del contenedor

Ahora que ya tenemos un entorno virtual activo, podemos descargar el código inicial para arrancar el proyecto para construir la imagen del contenedor

```
- pip install Django==4.2.10  
- cd \~/src/geonode/app  
- mkdir datosmty  
- GN\_VERSION=datos\_mty  
- django-admin startproject \--template=https://github.com/gobiernodigitalmonterrey/geonode-project/archive/refs/heads/$GN\_VERSION.zip \-e py,sh,md,rst,json,yml,ini,env,sample,properties \-n monitoring-cron \-n Dockerfile datosmty datosmty
```
A continuación debemos crear un archivo **.env** de variables de entorno para ello utilizaremos el script ***create-envfile.py** que está dentro del proyecto de django que acabamos de crear en el paso anterior, dentro del directorio datosmty

El siguiente paso se debe hacer dos considerando que debe haber dos instancias para uso público o interno, en este caso se debe hacer una vez hostname como ```mide.monterrey.gob.mx``` y otra como ```admide.monterrey.gob.mx```.

```
- cd datosmty 
- python3 create-envfile.py \--env\_type=prod \--hostname=mide.monterrey.gob.mx \--https \--email=gobierno.digital@monterrey.gob.mx
```

  *El último comando nos pedirá confirmar si queremos sobreescribir el archivo* **.env** *en caso de existir, confirmamos escribiendo la letra* \[y\]*, a continuación pulsamos la tecla* \[Enter\]*. A continuación se nos indica que el archivo* **.env**  *ha sido creado.*

  *Editar el archivo* **.env** *y revisar la clave* **SECRET\_KEY***, asegurarse que no contiene ningún carácter “$” como parte de la clave, de lo contrario puede generar errores. Esta clave se genera automáticamente y de manera aleatoria cada vez que se genera el archivo .env. Es importante conservar esta clave, pues si ya se tienen contenedores corriendo es necesario para acceder a los datos que se almacenan cifrados, pues se utiliza esta clave para hacer dicho cifrado, en caso de perder esta clave es imposible acceder a la información cifrada dentro de la base de datos, pues esta es la clave que se utiliza para el salteado de la información cifrada.*


## Base de datos

En una instalación predeterminada la configuración de bases de datos ya no se debe de tocar, pero en este caso vamos a conectar las bases de datos hacia una instancia de AlloyDB, para ello debemos crear dos bases de datos con su respectivo usuario con permiso de conexión a ellas, esto de acuerdo al documento de creación de bases de datos de AlloyDB. Por motivos de seguridad, vamos a asumir el hipotético caso de contar con un usuario llamado god\_datosmty con acceso a las bases de datos llamadas ```god\_datosmty\_django``` y  ```god\_datosmty\_data``` con el password **sp6CAiAlCW0ON9p** así como la IP de conexión ```10.100.192.2```. **Tener presente que estos datos son indicativos y se debe asegurar el tener los datos correctos reales, los cuales por seguridad no se pueden plasmar en este documento.**

De acuerdo a lo anterior, hay que modificar los parámetros de conexión de acuerdo al siguiente ejemplo, conforme los datos mencionados en el párrafo anterior. Estos ajustes hay que modificarlos en el archivo **.env** a partir de la línea 30 (se estilan en negritas los datos que se deben modificar).

```
POSTGRES\_USER=god\_datosmty  
POSTGRES\_PASSWORD=sp6CAiAlCW0ON9p  
GEONODE\_DATABASE=god\_datosmty\_django  
GEONODE\_DATABASE\_USER=god\_datosmty  
GEONODE\_DATABASE\_PASSWORD=sp6CAiAlCW0ON9p 
GEONODE\_GEODATABASE=god\_datosmty\_data  
GEONODE\_GEODATABASE\_USER=god\_datosmty  
GEONODE\_GEODATABASE\_PASSWORD=sp6CAiAlCW0ON9p
GEONODE\_DATABASE\_SCHEMA=public  
GEONODE\_GEODATABASE\_SCHEMA=public  
DATABASE\_HOST=10.100.192.2  
DATABASE\_PORT=5432  
DATABASE\_URL=postgis://god\_datosmty:sp6CAiAlCW0ON9p@10.100.192.2**:5432/god\_datosmty\_django  
GEODATABASE\_URL=postgis://god\_datosmty:sp6CAiAlCW0ON9p@10.100.192.2:5432/god\_datosmty\_data
```

## Ajustar .env

Hay que ajustar algunos parámetros adicionales dentro del archivo **.env**, de modo que las variables enlistadas a continuación tengan los valores aquí presentados.

Hay que tener especial atención en que al haber la necesidad de instalar dos instancias que apunten a la misma base de datos pero con distinta configuración de OIDC, es necesario ajustar cada caso particular relativo a ODIC.

### ID Digital Mty+

```
SOCIALACCOUNT\_OIDC\_PROVIDER\_ENABLED=True 
SOCIALACCOUNT\_PROVIDER=IDMty  
SOCIALACCOUNT\_OIDC\_PROVIDER=IDMty  
SOCIALACCOUNT\_PROVIDER\_CONFIG='{"NAME": "IDMty", "SCOPE": \["email", "profile", "openid"\], "AUTH\_PARAMS": {}, "COMMON\_FIELDS": {"email": "email", "name": "displayName"}, "ACCESS\_TOKEN\_URL": "https://iam.monterrey.gob.mx/realms/IDMty/protocol/openid-connect/token", "AUTHORIZE\_URL": "https://iam.monterrey.gob.mx/realms/IDMty/protocol/openid-connect/auth", "PROFILE\_URL": "https://iam.monterrey.gob.mx/realms/IDMty/protocol/openid-connect/userinfo", "OAUTH\_PKCE\_ENABLED": True}'                                     	   
SOCIALACCOUNT\_PROVIDER\_LOGOUT\_URL=https://iam.monterrey.gob.mx/realms/IDMty/protocol/openid-connect/logout

### ID Gob Mty+

SOCIALACCOUNT\_OIDC\_PROVIDER\_ENABLED=True
SOCIALACCOUNT\_PROVIDER=GobMty  
SOCIALACCOUNT\_OIDC\_PROVIDER=GobMty  
SOCIALACCOUNT\_PROVIDER\_CONFIG='{"NAME": "GobMty", "SCOPE": \["email", "profile", "openid"\], "AUTH\_PARAMS": {}, "COMMON\_FIELDS": {"email": "email", "name": "displayName"}, "ACCESS\_TOKEN\_URL": "https://iam.monterrey.gob.mx/realms/GobMty/protocol/openid-connect/token", "AUTHORIZE\_URL": "https://iam.monterrey.gob.mx/realms/GobMty/protocol/openid-connect/auth", "PROFILE\_URL": "https://iam.monterrey.gob.mx/realms/GobMty/protocol/openid-connect/userinfo", "OAUTH\_PKCE\_ENABLED": True}'                                     	   
SOCIALACCOUNT\_PROVIDER\_LOGOUT\_URL=https://iam.monterrey.gob.mx/realms/GobMty/protocol/openid-connect/logout
```

## Construir imágenes de los contenedores

Una vez tenemos docker instalado, el entorno virtual activo, el proyecto inicializado y el archivo de variables de entorno, ahora estamos listos para construir las imágenes para los contenedores, para ello continuamos con el siguiente comando, el cual va a tomar algunos minutos.

```
- sh docker-build.sh
```

Después de algunos minutos (aproximadamente 8 a 10 minutos), si todo ha salido bien, se nos mostrará al final la siguiente información:  
[imagen_1]

En este punto, se nos pregunta si deseamos continuar, pulsamos la tecla \[y\] para continuar, casi hemos terminado, pero aún debemos esperar algunos minutos a que se hagan algunos procesos como el poblado inicial de las bases de datos y se levanten algunos servicios.

A continuación podemos ver el estatus de los contenedores con el comando:

```
- docker container ps \--format "table {{.ID}}\\t{{.Image}}\\t{{.RunningFor}}\\t{{.Status}}\\t{{.Names}}"
```
El cual nos presenta la siguiente tabla:
[imagen_2]

En donde podemos ver distinta información relativa a cada contenedor, presentada en una tabla, donde cada fila es un contenedor, en la columna STATUS podemos ver el estado actual, podremos ver que muestran estado “Up” y el tiempo que llevan corriendo, además entre paréntesis se muestra su estado de salud en algunos, el estado “healthy” significa que están corriendo correctamente, en caso de observar estado “health starting” significa que aún está iniciando el contenedor y que hasta ahora todo va correcto pero no ha terminado de levantar por lo que debemos esperar unos pocos minutos más, podemos ejecutar el mismo comando que nos presenta esta tabla eventualmente, hasta que veamos todos los estados como healty en los cuatro casos que lo señala.

A continuación se muestra el resultado de volver a hacer la consulta del estado de los contenedores, donde podemos ver que todos están corriendo y en los casos que se monitorea el estado ya todos dicen “healthy”
[imagen_3]

Es importante señalar que el **CONTAINER ID**, es un identificador único, que se genera cada vez que se levanta un contenedor, por lo que los IDs con toda seguridad serán distintos. En la columna names se utiliza un nombre que está compuesto por el nombre de la imagen base seguido de un **4** y el nombre del proyecto que se creó varios pasos antes con el comando django-admin, en este caso “**datosmty**”; por lo que vemos nombres como “**geoserver4datosmty**”, lo cual indica que es un contenedor basado en geoserver del proyecto datosmty.

# Configuración de OIDC

Para completar la configuración de OpenID Connect para integrar **ID Mty+**, hay que acceder como administrador primero en el método nativo, para esto hay que ir a la url del dominio seguido de ```/admin/``` y usar el usuario *admin* y el password asignado al crear el archivo **.env.**

Una vez dentro del *admin*, ir a la sección Cuentas de redes sociales \> Aplicaciones de redes sociales y añadir una aplicación de redes sociales. Hay que tener especial cuidado en este apartado, esto se debe hacer una sola vez para cada sitio, y el caso respectivo para cada sitio debe hacerse desde dentro del admin de su propio sitio, aunque se vean ambas aplicaciones en ambos sitios al compartir la base de datos, es muy importante que solo se haga en su respectivo sitio o puede causar errores.

Debemos llenar los datos de acuerdo a lo siguiente:

- `Proveedor`: Seleccionar el disponible (depende del sitio actual, por eso la nota previa)  
- `Provider ID`: el client\_id designado para este uso en keycloak (por ejemplo datosmty.monterrey.gob.mx)  
- `Nombre`: Escribir el mismo nombre que el campo seleccionado en Proveedor  
- `Identificador cliente`: el mismo contenido que en Provider ID, es decir, el client\_id   
- `Clave secreta`: Secret key de keycloak  
- `Clave`: dejar vacío  
- `Sites`: pasar al lado derecho el dominio del sitio actual

De modo indicativo y de ejemplo, se debería ver como la siguiente imagen:
[imagen_4]

## Ajustes adicionales de GeoServer
Para permitir descargar archivos grandes hay que modificar la configuración de WPS en geoserver, para ello hay que acceder a geoserver como administrador, en la sección Servicios \> WPS, casi al final cambiar los valores de WPS Download como se muestra a continuación:

[imagen_5]



