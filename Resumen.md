
# Docker files

Crear un app a partir de un Dockerfile

Docker Instructions

    FROM

    RUN

    LABEL

    ENV

    USER

    VOLUME

    ONBUILD

Adapting Docker File for Openshift

    RUN chgrp -R 0 directory && \
        chmod -R g=u directory
    
Administrando imagenes en el registro

Uso de podman y Skopeo

    podman pull
    podman push

    skopeo copy
    skopeo delete
    skopeo inspect
  
## Push & Tagging Images

### Desde el cache local

    skopeo copy --dest-tls-verify=false \
     containers-storage:myimage \
     docker://registry.example.com/myorg/myimage

### Desde archive

    skopeo copy \
     docker-archive:php-70-rhel7-original.tar.gz \
     docker://quay.io/${RHT_OCP4_QUAY_USER}/php-70-rhel7:latest

### Desde formato oci

    skopeo copy --dest-tls-verify=false \
     oci:/home/user/myimage \
     docker://registry.example.com/myorg/myimage
     
### Entre registros privados

    skopeo copy --src-creds=testuser:testpassword \
     --dest-creds=testuser1:testpassword \
     docker://srcregistry.domain.com/org1/private \
     docker://dstegistry.domain2.com/org2/private

### Copiar y tagear imagen en repositorio destino

    skopeo copy docker://registry.example.com/myorg/myimage:1.0 \
     docker://registry.example.com/myorg/myimage:latest
     
### Borrar Imagen de un registro:

    skopeo delete docker://registry.example.com/myorg/myimage
    
## Authenticacion contra un registro privado

Tipo de Imagenes:

- **oci**: denotes container images stored in a local, OCI-formatted folder.
- **docker**: denotes remote container images stored in a registry server.
- **containers-storage**: denotes container images stored in the local container engine cache.

Se crea el secret

    oc create secret docker-registry registrycreds \
    --docker-server registry.example.com \
    --docker-username youruser \
    --docker-password yourpassword

Enlazar el secret a la cuenta default para que pueda hacer pull del registro:

    oc secrets link default registrycreds --for pull

Enlazar el secret a la cuenta builder para que pueda hacer pull durante la construccion S2I

    oc secrets link builder registrycreds

## Problemas que podriamos enfrentar:

- **Problema**: Tratamos de usar una imagen de un registro privado en el cual necesitamos autenticarnos.
- **Solucion**: Creamos un secret en el proyecto y lo linkeamos a la cuenta de sevicio (service account) del **builder** o el **default**.

# Exponiendo el Registro
> Documentacion Redhat: Seccion Registry: Cap. 6.

La configuracion del registro interno de openshift se encuentra dentro del proyecto **openshift-image-registry** en el recurso **cluster**

Para exponer el registro interno debemos habilitar la especificacion: **spec.defaultRoute**, osea que debemos pasarlo de false a **true**.

El camino dificil es recordar el siguiente comando:

    oc patch config cluster -n openshift-image-registry \
    --type merge -p '{"spec":{"defaultRoute":true}}'

El camino facil es editar el recurso manualmente:

    oc edit config cluster -n openshift-image-registry
    
## Autenticando contra el registro interno usando podman o skopeo.

Para realizar la autenticacion contra el registro de openshift, solo se puede a travez del token del usuario. Este token lo obtenemos a traves del siguiente comando:

    oc whoami -t
    
    Truco: Podemos guardar la variable usando:
    
    TOKEN=$(oc whoami -t)
    
Ahora para autenticarse con podman:

    podman login -u myuser -p ${TOKEN} default-route-openshift-image-registry.domain.example.com

    skopeo inspect --creds=myuser:${TOKEN} docker://default-route-openshift-image-registry.domain.example.com/
    
## Ubicar la URL del registro

    oc get route -n openshift-image-registry

## Cuando acceder al registro de forma insegura o insegura.

Cuando el certificado del wildcard de Openshift usa una entidad certificadora publica.

    skopeo inspect docker://default-route-openshift-image-registry.domain.example.com/myproj/myapp

Cuando el certificado del wildcard de Openshift usa la entidad certificadora generada por el instalador, hay que indicarle que nos conectaremos de manera insegura. O que no verificaremos el certificado.

    skopeo inspect --tls-verify=false docker://default-route-openshift-image-registry.domain.example.com/myproj/myapp
    
## Permisos especiales de acceso a imagenes del registro
> General OpenShift authentication and authorization concepts are outside the scope of this course. Refer to Red Hat Training courses on the OpenShift administration track, such as Red Hat OpenShift Administration I (DO280) and Red Hat Security: Securing Containers and OpenShift (DO425), for more information about configuring TLS certificates for OpenShift and your local container engine.

Por defecto Openshift permite a todo aquel que tiene permisos **admin** o **edit** en un proyecto hacer **pull** y **push** de las imagenes del registro. Los que tienen permiso **view** no pueden. Sin embargo hay casos en que se requiere dar permisos especificos para realizar pull y push. Para ello se crearon los siguientes roles:

**registry-viewer** and **system:image-puller**

**registry-editor** and **system:image-pusher**

### Agregar permisos a usuario para hacer pull de imagenes del registro hacia un proyecto especifico:

    oc policy add-role-to-user system:image-puller user_name -n project_name
    oc policy add-role-to-user system:image-pusher user_name -n project_name

# Image Streams
> Nota Importante: It is outside the scope of this course to teach all aspects of image stream management, although
some of them, such as image change events, are explored in later chapters. Pag 150

## Administrando Image Stream Tags

Para crear un image stream solo tenemos que importar la info de una imagen. Para ello se utiliza el comando **oc import-image **

    oc import-image myimagestream[:tag] --confirm --from registry/myorg/myimage[:tag]

Si no se usa el **[:tag]** openshift le asignara **latest**

>To create an image stream for container images hosted on a registry server that is not set up with a trusted TLS certificate, add the --insecure option to the oc import-image command.

Para crear un is que contemplen todos los tags de la imagen fuente utilizamos la opcion **--confirm --all**

    oc import-image myimagestream[:tag] --confirm --all --from registry/myorg/myimage[:tag]

### Actualizar un IS existente en el registro

Para **actualizar** un image stream del repositorio:

    oc import-image --confirm

Para **actualizar** un image stream del repositorio con todas sus versiones inclusive si tiene nuevas:
    
    oc import-image --confirm --all
    
### Compartir IS entre Proyectos    

La mejor manera de compartir un IS entre proyectos es crear un proyecto para que contenga el IS y que todo los demas proyectos puedan acceder a este para hacer uso del IS. Para ello se debe hacer lo siguiente:

1. Authenticarse contra el registry que se desea utoilizar

        podman login -u myuser registry.example.com
        
2. Crear un proyecto para compartir

        oc new-project shared

3. Crear un secret con las credenciales del registro

        oc create secret docker-registry registrycreds \
        --docker-server registry.example.com \
        --docker-username youruser \
        --docker-password yourpassword
        
4. Se crea el IS en el proyecto importandolo.

        oc import-image myis --confirm 
        --reference-policy local 
        --from registry.example.com/myorg/myimage

> *--reference-policy local* se usa para que openshift cachee localmente la imagen en el registro para que las app que la usen no tengan que ir al buscarla en el registro privado. 

5. Se le asigna el role de puller al service account

        oc policy add-role-to-group system:image-puller system:serviceaccounts:myapp

> Importante: *system:serviceaccounts:myapp* es un grupo de sistema que contempla todas las service account que existen en el proyecto *myapp*
> Importante: el comando **oc policy** puede referirse a grupos o usuarios que no se han creado todavia.
 
6. Se crea el proyecto donde se va usar el IS.

        oc new-project myapp
        
7. Se crea la app usando el IS.

        oc new-app --as-deployment-config -i shared/myis 

        oc new-app --as-deployment-config -i shared/myis~giturl      <---- Puede usarse para referenciar una imagen de construccion.

> Para referenciar la imagen se usa el {proyecto compartido}/{IS}

# Proceso de Construccion (Building Config process)

### Building Strategies

    Source: Codigo Fuente

    Pipeline: Jenkins

    Docker: Docker File

    Custom: Permite que desarrollador customice el proceso.

### Comandos utiles

    oc start-build name
    oc cancel-build name
    oc delete bc/name
    oc delete bc/name-1
    oc describe bc name
    oc logs -f bc/name
    
### Limitar el historico de los BC     

Usando los atributos:

- successfulBuildsHistoryLimit
- failedBuildsHistoryLimit


    apiVersion: "v1"
    kind: "BuildConfig"
    metadata:
      name: "sample-build"
    spec:
      successfulBuildsHistoryLimit: 2
      failedBuildsHistoryLimit: 2

### Hacer un purgado automatico

Solo los administradores con el comando:

    oc adm prune
    
> Sometimes you may need to ask administrators to change the pruning configuration so that build logs are retained longer than the default so that you can troubleshoot build issues. These topics are out of scope for the current course   

### Incorporar variables durante el proceso de contruccion con el comando *oc

Usando el parametro **--build-env**

    oc new-app --as-deployment-config --name jhost \
     --build-env MAVEN_MIRROR_URL=http://${RHT_OCP4_NEXUS_SERVER}/repository/java \
     -i redhat-openjdk18-openshift:1.5 \
     https://github.com/${RHT_OCP4_GITHUB_USER}/DO288-apps#manage-builds \
     --context-dir java-serverhost

## Triggers 

Son disparadores de acciones que se definen en openshift al detectarse algun cambio.

### Tipos de Triggers

- **Image Change triggers**: Reconstruye una imagen ante un cambio en la imagen utilizada.
- **Webhook triggers**: Son endpoints que estan a la escucha preparados para detectar un evento y disparar una accion. En este caso disparar un **build**.

### Como visualizar los triggers existentes en un BC

    oc describe bc/name

### Como configurar un trigger de cambio de imagen en un BC

    oc set triggers bc/name --from-image project/image:tag
    
> OJO: Un BC no puede tener multiples triggers de cambio de image.

### Como elimino el trigger de cambio de imagen en un BC (porque solo debe haber uno solo)

    oc set trigger bc/name --from-image project/image:tag --remove 

# Post-Commit Build Hooks

Este Hook se ejecuta justo despues de terminar el proceso build. Este fue pensado para realizar validaciones posteriores a la terminacion del proceso. Para ello, el hook arranca un contenedor temporal con la imagen recien creada por el build.

Como es basicamente un proceso de validacion de la imagen. Si ese proceso no temina exitosamente, la imagen no se publica en el registro.

## Como configurar un Build Hook


Tipo **Command**:

    oc set build-hook bc/name --post-commit \
     --command -- bundle exec rake test --verbose
 
>The space right after the -- argument is not a mistake. The -- argument separates the OpenShift command line that configures a post-commit hook from the command executed by the hook. 

Tipo **Shell Script**:

    oc set build-hook bc/name --post-commit \
     --script="curl http://api.com/user/${USER}"

# Source to Image (S2I)

Esta compuesto basicamente de 3 componentes:

- Source code: Codigo Fuente
- S2I scripts: Los scripts utilizado por el S2I
- S2I Builder Image: La image builder a usar.

## S2I Scripts

- Assemble (requerido): son los scripts de ensamblaje, para colocar el fuente en los directorios correctos dentro de la imagen.
- run (requerido): Es el script de ejecucion de la aplicacion. Se recomienda usar el comando exec.
- save-artifacts (no requerido): Son los scripts que colectan las dependiencias y las comprime en un tar.
- usage (no requerido): Indica como se usa la imagen
- test/run (no requrido): Scripts para verificacion de la imagen.

## Custoizar imagen S2I existente

Basicamente debemos agregar nuestros propios scripts dentro de la carpeta **.s2i/bin** Durante el proceso de construccion los nuevos scripts son detectados y son ejecutados en vez de los scripts originales del S2I.

OJO: El codigo fuente es copiado a **/tmp/src**

1. Debemos hacer un pull de la imagen que se desea modificar.

        sudo podman pull myregistry.example.com/rhscl/php-73-rhel7

2. Inspeccionar la imagen y verificar el valor del atributo **io.openshift.s2i.scripts-url**

*Primer metodo: se inspecciona localmente

    sudo podman inspect --format='{{ index .Config.Labels "io.openshift.s2i.scripts-url"}}' rhscl/php-73-rhel7
    
    El comando retornara la ubicacion:
    
    image:///usr/libexec/s2i

*Segundo metodo: se inspecciona remotamente

    skopeo inspect \
    > docker://myregistry.example.com/rhscl/php-73-rhel7 \
    > | grep io.openshift.s2i.scripts-url
    "io.openshift.s2i.scripts-url": "image:///usr/libexec/s2i",

3. Modificar el fichero **.s2i/bin/assemble**
> Esta carpeta generalemente esta ubicada en /usr/libexec/






