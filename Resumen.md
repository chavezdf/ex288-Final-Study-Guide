
## Docker files

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

## Permitiendo el acceso al registro interno de Openshift

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

### 
