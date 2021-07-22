
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

## Customizar imagen S2I existente

Basicamente debemos agregar nuestros propios scripts dentro de la carpeta **.s2i/bin** en nuestro codigo fuente. Durante el proceso de construccion los nuevos scripts son detectados y son ejecutados en vez de los scripts originales del S2I.

Tip: El codigo fuente es copiado a **/tmp/src**

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

3. Se puede hacer un wrapper del assemble original y colocarlo en la carpeta **.s2i/bin/** en nuestro codigo
> La carpeta en la imagen generalemente esta en: /usr/libexec/

## Crear una Imagen de Construccion
> OJO: Esta seccion no esta contemplada dentro de los objetivos del examen, por lo tanto no contemplamos el proceso completo de creacion de una image de construccion.
> OBJECTIVES: Work with the source-to-image (S2I) tool
> - Deploy applications using S2I
> - Customize existing S2I builder images


# Templates

Los templates estan basados en la definicion de RECURSOS compilados como OBJETOS, los cuales son personalizados a traves de la definicion de PARAMETROS.

## Parametros 

Los parametros son llamados en los recursos utilizando ${PARAMETRO} y son definidos en el bloque parameters dento del template. Estos pueden ser definidos como obligatorios y requeridos y tambien puede proveer valores por defecto.

    parameters:
    - description: Myapp configuration data         
    name: MYPARAMETER                               <-- nobmre de parametro
    value: /etc/myapp/config.ini                    <-- parametro por defecto

    parameters:
    - description: ACME cloud provider API key
    name: APIKEY
    generate: expression                            <-- de tipo expresion generada automaticamente
    from:"[a-zA-Z0-9]{12}"                          <-- coo se va generar
    
### Agregar templates en openshift

    oc create -f template.yaml -n proyecto 
    
### Visualizando un template   

**Listar:**

    oc get templates -n openshift
    
**Listar parametros:**
    
    oc process --parameters -n openshift nodejs-mongodb-example

**Descargar template a fichero:**

    oc get template nodejs-mongodb-example -o yaml -n openshift

## Creando Templates

### Ideas para iniciar:

Podemos utilizar **oc explain** y **oc api-resources** para ver los atributos de cada tipo de recurso que se desea personalizar.

### Procedimiento general

1. Crear un esqueleto de un template con los atributos como NAME, TAGS, y METADATOS Necesarios
2. Utilizar el comando oc get -o --export {recursos} para exportar a fichero

        oc get -o yaml --export is,bc,dc,svc,route > mytemplate.yaml

3. Limpiar los atributos de tiempo de ejecucion

        Atributos o secciones como: status, creationTimeStamp, image, uid, etc

4. Cortar y pegar los recursos al eseuleto del template previamente creado.

### Problemas a considerar

- No es posible inicializar loca valores KEY: VALUE dentro del atributo DATA utlizando un PARAMETRO. Este atributo debe ser reemplazado por un atributo stringData, y todos los key value deben estar descodificados.

- Si un Image Stream estan vinculados a un registro externo. Entonces se debera cambiar en la plantilla el IS que apunte al externo porque sino fallara.

### Herramientas para la exploracion de definicion de recursos

    oc explain routes

    oc explain routes.spec
    
    oc api-resources

### Crear una applicacion a partir de un template

Visualizar parametros a utliizar:

    oc process -f mytemplate.yaml --parameters

    oc process --parameters mytemplate -n proyecto

Forma rapida:

    oc new-app --file mytemplate.yaml -p PARAM1=value1 -p PARAM2=value2

Exportando el template con parametros definidos:

    oc process -f mytemplate.yaml -p PARAM1=value1 -p PARAM2=value2 > myresourcelist.yaml

    oc create -f myresourcelist.yaml
    
Proceso 2 en 1:

    oc process -f mytemplate.yaml -p PARAM1=value1 -p PARAM2=value2 | oc create -f -
    
# Monitoreo de Aplicaciones

Openshift proporciona 2 formas de verificar el buen funcionamiento de las aplicaciones:

**Readiness Probe**: Verifica si la app ya esta lista para responder a peticiones 

**Liveness Probe**: Verifica si app esta viva. Esta se ejecuta primero de el Readiness.

### Parametros que usan

- **initialDelaySeconds:** Tiempo de espera para que la app inicie y comenzar el probe.
- **timeoutSeconds:** Tiempo maximo a esperar para la respuesta del probe.
- **periodSeconds:** Cada cuando se ejecuta el probe
- **successThreshold:** Cantidad de intentos exitosos del probe luego de un fallido, para considerar que la app vuelve a estar bien. 
- **failureThreshold:** Cantidad minima de intentos fallidos del probe para considerar que la app fallo.

### Metodos que se pueden usar para la verificacion de la salud de la aplicacion

**HTTP Checks:** Usa los **HTTP STATUS CODES**, ideal para las APIs

**Ej: Definicion de Recursos:**

    readinessProbe:
      httpGet:
        path: /health
        port: 8080
    initialDelaySeconds: 15
    timeoutSeconds: 1

**Ej: Commando:**

    oc set probe dc/myapp --readiness -get-url=http://:8080/healthz --period=20

**Container Execution Checks:** Usa el **EXIT CODE** del proceso o el script utilizado para inicial la app dentro del contenedor

**Ej: Definicion de Recursos:**

    livenessProbe:
      exec:
        command:
          - cat
          - /tmp/health
    initialDelaySeconds: 15
    timeoutSeconds: 1

**Ej: Commando:**

    oc set probe dc/myapp --liveness --open-tcp=3306 --period=20 --timeout-seconds=1

**TCP Socket Checks:** Usa los **PUERTOS TCP** para verificar si la app o demonio esta en ejecucion. Tradicionalmente es usado para demonios de bases de datos, file servers, webserver y application servers.

**Ej: Definicion de Recursos:**

    livenessProbe:
    tcpSocket:
      port: 8080
    initialDelaySeconds: 15
    timeoutSeconds: 1
    succuessThreshold: 1
    failureThreshold: 3

**Ej: Commando:**

    oc set probe dc/myapp --liveness \
     --get-url=http://:8080/healthz --initial-delay-seconds=15 \
     --succuess-threshold=1 --failure-threshold=3
     
# Deplolyment Strategies

Se utilizan para definir el metodo que se va a utilizar para actualizar o combiar una aplicacion. El objetivo de estas estrategias es realizar los cambios en el menor tiempo, reduciendo el impacto.

Los metodos son:

**Rolling:** *Es la por defecto*. Reemplaza las instancias una a una progresivamente.
**Recreate:** Openshift detiene todos los pods, para luego desplegar la nueva version. Se recomienda, cuando correr la nueva version y la vieja no es compatible.
**Custom:** Cuando ninguno de los 2 metodos anterior cumplen el objetivo y necesitas personalizar ciertos comandos.

## Life-cycle Hooks

Son hooks que se ejecutan durante el deployment. Estos son:

**Pre-Lifecycle Hook:** Se ejecuta antes de que termine alguno de los viejos PODs y antes de comenzar cualquiera de los nuevos PODs

**Mid-Lifecycle Hook:** Se ejecuta despues de apagar cualquiera de los PODs viejos pero antes de comenzar cualquiera de los nuevos PODs

**Post-Lifecycle Hook:** Se ejecuta despues de comenzar el nuevo POD pero luego de haber apagado TODOS los viejos PODs

Cada uno tiene policitas de fallo (failurePolicy):

• Abort: The deployment process is considered a failure if the hook fails.
• Retry: Retry the hook execution until it succeeds.
• Ignore: Ignore any hook failure and allow the deployment to proceed.

