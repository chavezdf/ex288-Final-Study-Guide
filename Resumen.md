Trabajando com Imagenes del Registro


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

Authenticacion contra un registro privado

Tipo de Imagenes:

**oci**: denotes container images stored in a local, OCI-formatted folder.
**docker**: denotes remote container images stored in a registry server.
**containers-storage**: denotes container images stored in the local container engine cache.

Se crea el secret

    oc create secret docker-registry registrycreds \
    --docker-server registry.example.com \
    --docker-username youruser \
    --docker-password yourpassword

Enlazar el secret a la cuenta default para que pueda hacer pull del registro:

    oc secrets link default registrycreds --for pull

Enlazar el secret a la cuenta builder para que pueda hacer pull durante la construccion S2I

    oc secrets link builder registrycreds
    
    
