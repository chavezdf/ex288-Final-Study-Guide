
## Capitulo 2 
	Guiado 1:
		Objetivo:
			- Usar una imagen Padre (Con declaraciones ONBUILD) para construir una imagen Hija.
		Troubleshoot:
			- Hay que inspeccionar la imagen padre, y tener claro que le esta permitiendo hacer al hijo.
			- Si el comando ONBUILD especifica una copiando de una carpeta, entonces hay que asegurarse que la carpeta donde se este ejecutando el Dockerfile
			del hijo contenga la carpeta o los ficheros que se van a copiar.
	Guiado 2:
		Objetivo:
			- Inyectar data de configuracion a un contenedor
			- Cambiar data de configuracion
	LAB Capitulo 2:
		Objetivo:
			- Arreglar una Dockerfile para que pueda ejecutarse con un usuario aleatorio, y pueda ejecutarse correctamente en un entorno de OCP.
			- Usar Config Maps y Secret para agregar data de configuracion
		Troubleshoot:
			- OJO: Hay que estar pendiente de la URL completa, donde se supone que va responder algo. No es solo el route solo. aveces le agregan endpoints.

## Capitulo 3 
	Guiado 1:
		Objetivo:
			- Subir imagen a un registro externo, autenticando contra el registro.
			- Desplegar una aplicacion en OCP usando una imagen externa que requiera autenticacion con entrada.
		Troubleshoot:
			- probar imagen externa: podman run -d --name test <path-to-image> -p <port>:<port>		
			- crear secret con creadenciales
			- crear link a la SA builder o default: oc secret link <serviceAccount> <secret-name>

	Guiado 2:
		Objetivo:
			- Subir imagen a in registro interno usando las herramientas de contenedores (Exponer el Registro Interno)
			- Crear un contenedor usando el registro interno
		Troubleshoot:
			- Para logearte al registro interno, puedes usar el metodo expresado en la documentacion (Seccion: Registry Cap. 6)
			- Tambien se puede usar la opcion --dest-cred de skopeo. Colocando user:token
			- OJO: la ruta destino del registro interno es: docker://<nom-registro>/<nom-proyecto>/<nom-imagen>:<tag>
			- Luego de subir la imagen, debemos verificar que se puede baja con podman pull.

		Comandos relevantes:
			Informacion necesaria:
				TOKEN=$(oc whoami -t)
				oc get route -n openshift-image-registry
				OJO: el nombre por defecto "default-route"-"openshift-image-registry"."wildcard"		
			Autentication: USER:TOKEN			
				skopeo copy --dest-creds <usuario>:<token> <imagen-origen> docker://<nobmre-registry>/<nobmre-proyecto>/<nombre-image>:<tag>
			TEST: 
				podman login -u <user> -p <token>
				podman pull <nombre-registro>/<nombre-proyecto>/<nobmre-image>:<tag>
				podman run  --name <nombre> <nombre-registro>/<nombre-proyecto>/<nobmre-image>:<tag>
	Guiado 3:
		Objetivo:
			- Publicar una image de un registro externo (NO PRIVADO) en un proyecto y hacerla accesible a otros proyectos.
			- Desplegar una aplicacion usando esta imagen.
		Troubleshoot:
			- Creamos un proyecto de uso comun
			- Importamos la imagen como IS en el proyecto comun. OJO: usar oc import-image <nom-is> --from <image-url> 
			- Se crea un nuevo proyecto donde se va hacer uso del IS importado previamente.
			- Se invoca al IS: oc new-app --as-deployment-config --name <nombre> -i <nombre-project>/<nombre-de-IS>
			OJO: Como la imagen se importo desde un registro publico, los proyectos no va a tener problemas para hacer pull.
		
	LAB 3:
		Objetivo:
			- Desplegar una aplicacion haciendo uso de una imagen de un registro externo.
		Troubleshoot:
			- Se sube una image OCI a un registro PRIVADO externo.
			- Se crea un proyecto de uso comun
			- Para importar la imagen al OCP, al ser un registro privado, se deben crear un secret con las credenciales.			
			- Luego de importar la imagen, se crea un nuevo proyecto para realizar uso de la imagen.
			- Para ello, se debe agregar el grupo **system:image-puller** al grupo **system:serviceaccounts:<nombre-de-proyecto>** 
			en el namespace del proyecto de uso comun common, con la finalidad de poder realizar pull.
			- Finalmente, podemos crear una aplicacion usando la imagen del proyecto de uso comun.

		Comandos Relevantes:
			oc import-image <nombre-imgage>:<tag> --from <path-to-registry-image> --confirm --reference-policy local
			oc secret link <service-account> <secret> --for <pull,push,mount>
			oc policy add-role-to-group system:image-puller system:serviceaccounts:<nobmre-proyecto>
			oc new-app --name <nombre-app> --as-deployment-config <nombre-proyecto-comun>/<nombre-image-stream>

## Capitulo 4

	Guiado 1: 
		Objetivo: 
			- Manejar builds
		Troubleshoots:
			- Problemas con variables de entorno de construccion BUILD_ENV
		Comandos relevantes:
			oc start-builde bc/<nombre-build>
			oc cancel-build bc/<nombre-build>			

	Guiado 2: 
		Objetivo: 
			- Desplegar una aplicacion con una imagen de contruccion
			- Actualizar version de imagen de contruccion y verificar se dispara un trigger por cambio en la imagen.
		Troubleshoot:
			- Cuando se esta importando una imagen de un registro privado donde es necesario el acceso con credenciales
			se debe hacer el link del "secret" con la seviceaccount "builder"
		Comandos relevantes:
			oc secret link builder <nombre-secret>
			oc import-image IS  (Para actualizar el IS ante cambio)

	Guiado 3: 
		Objetivo: 
			- Crear un commando Post-Commit
		TroubleShoot: 
			- Una de las variables utilizada en el comando post commit no estaba definida como variable de entorno.
		Commando relevante: 
			oc set build-hook bc/name --post-commit --command -- COMANDO
			oc set build-hook bc/name --post-commit --script="SCRIPT"
			oc set env bc/name VARIABLE=valor
			oc set env bc/name --list			

	LAB: Capitulo 4

		Objectivo:
			- Verificar un BuildConfig.
			- Ejecutar un BuildConfig utilizando un webhook generico.
		TroubleShoot: 
			- Una variable de entorno estaba mal definida.

		Commandos relevantes:

			oc set env bc/name VARIABLE=valor
			oc set env bc/name --list


## Capitulo 5:

	Guiado 1:

		Objetivo:
			- Personalizar un "assemble" y correr un script de una imagen de construccion de Apache. 
			- Sobreescribir el script por defecto de la imagen de construccion.
		A tomar en consideracion:
			- Las imagenes cuando arrancan se posicionan en el directorio "/opt/app-root/src". Este directorio es donde se instala el fuente.
			- Por ello los assemble hacen una copia desde /tmp/src/* ./
			- Cuando se quiere personalizar una imagen de construccion, se debe agregar en el codigo fuente los directorios .s2i/bin
		Troubleshoot: 		
			- Inspeccionar la imagen con podman
			- Se debe crear un branch en el codigo para colocar estos ficheros en .s2i/bin
			- Se realizan las modificaciones necesarias a estos ficheros
			- Se crea una nueva app para probar que toma los cambios
			- Se verifican los logs de BC para visualizar los mensajes dejados por los nuevos scripts
			- Se verifican que los cambios hayan sido aplicados.

		Commandos relevantes

			podman run -it rute-imagen bash
			skopeo inspect ruta-imagen |grep s2i

			oc new-app --name --as-deployment-config istag~https://git-path-to-s2i-proyect --contect-dir directorio-s2i-project
			oc logs -f bc/nombre
	Guiado 2:
		Objetivos:
			- Construir una imagen de construccion usando s2i
			- Verificar la image S2I localmente
			- Publicar imagen de construccion en Quay.io
			- Desplegar y probar la imagen de construccion en OCP.
		Troubleshoot:
			- Generar directorio con ficheros S2I usando "s2i create"
			- En el directorio Creado:
				- Crear o sustituir el Dockerfile
				- Eliminar el archivo s2i/bin/save-artifacts si no se va usar.
			- Generar la imagen de contruccion con sudo podman build -t name .
			- Probar la imagen con codigo de prueba con s2i
			- logearse en el registry privado
			- Subir imagen con skopeo desde el containers-storage
			- Crear secret con credenciales del registro en proyecto
			- Importar imagen al proyecto
			- Probar imagen creando una aplicacion
			
		Comandos relevantes:
			Generar estructura de directorio s2i
				s2i create nom-image nom-directorio
			Al tener el Dockerfile listo:
				Construir imagen de construccion con nombre img-Name
					sudo podman build -t img-Name . 
			Generar Dockerfile para probar imagen de construccion con con codigo fuente de prueba
				s2i build source-code image-de-construccion-podman-path img-Name --as-dockerfile dest-path-to-dockeerfile 
			Construir imagen usando la imagen de construccion y codigo fuente de prueba.
				sudo podman build -t img-test-name .
			Correr imagen resultante para verificarla
				sudo podman run -d -it --name test-name -u uid -p port:port img-test-name
			Copiar imagen a registro privado
				skopeo copy transport:full-name docker://private-registry:tag
				EJ: skopeo copy container-storage:localhost/image-name docker://private-registry:tag			
			Crear secret con credenciales
				oc create secret docker-registry nom-secret --docker-server=registry --docker-user=user --docker-password=password
			Linkear credenciales con el SA builder o default
				oc secrets link service-account-name secret-name
			Importar imagen
				oc import-image private-registry --confirm

	Lab: 
		Objectivo:
			- Contruir una imagen de construccion
			- Personalizar una imagen de construccion
			- Testear imagen de construccion creando un Dockerfile unsando la herramient s2i y crearla y correrla con podman
				-OJO: solicitan correr el contenedor con un usuario especifico con la opcion -u de podman			
			- Publicar la imagen creada a quay.io
			- Usarl la imagen de construccion creada para desplegar una aplicacion.
			- Personlizar los scripts nuevamente y redesplegar la imagen y probarla.
		Troubleshoot:
			- Se nos indica usar la imagen ubi8.0 o redhat8
			- Se nos da un Docker file ya generado para solo agregar el copiado del directorio s2i fuente
				COPY ./s2i/bin/ /usr/libexec/s2i
			- Generamos la imagen de construccion en local con podman.
			- Probamos codigo fuente usando la imagen generada utilizando la herramienta s2i.
			- Subimos la imagen de construccion creada y probada a quay.io
			- Creamos un secret con las credenciales de acceso al registry privado
			- Linkeamos el secret con las credenciales a la SA builder para que pueda hacer pulls
			- Importamos el IS de quay.io
			- Probamos el IS con un codigo fuente en OCP.
			- Probamos la aplicacion recien desplegada
			- Modificamos el script "run" de la imagen de construccion.
				- Agregamos en el fuente la carpeta .s2i/bin con el fichero run
				- Guardamos los cambios en Git
			- Realizamos un start-build otra vez.
		
		Commandos relevantes:
		
			Construir imagen de construccion con nombre img-Name
				sudo podman build -t img-Name . 
			Generar Dockerfile para probar imagen de construccion con con codigo fuente de prueba
				s2i build source-code image-de-construccion-podman-path img-Name --as-dockerfile dest-path-to-dockeerfile 
			Construir imagen usando la imagen de construccion y codigo fuente de prueba.
				sudo podman build -t img-test-name .
			Correr imagen resultante para verificarla
				sudo podman run -d -it --name test-name -u uid -p port:port img-test-name
			Copiar imagen a registro privado
				skopeo copy transport:full-name docker://private-registry:tag
				EJ: skopeo copy container-storage:localhost/image-name docker://private-registry:tag			
			Crear secret con credenciales
				oc create secret docker-registry nom-secret --docker-server=registry --docker-user=user --docker-password=password
			Linkear credenciales con el SA builder o default
				oc secrets link service-account-name secret-name
			Importar imagen
				oc import-image private-registry --confirm
			
## Capitulo 6: Template

	Guiado 1:
		Objetivos:
			- Crear un template desde una aplicacion en ejecucion
			- Limpiar los datos de tuntime de un template.
			- Agregar parametros a un template.
			- Crear una aplicacion desde un template.
		
		Troubleshoot:
			- Verificar que no hayan namespace apuntando al proyecto de donde viene el recurso
			
	Lab:
		Troubleshoot:
			- Los valores en el key "value" de los parametros debe estar entra commilas dobles o simples.

## Capitulo 7: Monitoreo de Aplicaciones

	Guiado 1:
		Objetivo:
			- Activar probes de liveness y readiness
			- Encontrar los mensajes de fallos
		
		Troubleshoot:
			- Las aplicaciones deben tener unas api que verifiquen si la aplicacion esta arriba y sirviendo.
			- se debe probar el endpoint
			- Se agrega el probe de liveness
			- Se agrega el probe de readiness
			- Se monitorean los logs
		
		Comandos Relevantes:
		
			oc set probe dc/<nobmre-DC> --readiness --get-url=http://:8080/healthz --initial-delay-seconds=2 --timeout-seconds=2
			
			readinessProbes:
			  httpGet:
			    path: /healthz
			    port: 8080
			  initial-delay-seconds: 2
			  timeout-seconds: 2
			    
			
			oc set probe dc/<nobmre-DC> --liveness  --get-url=http://:8080/ready --initial-delay-seconds=2 --timeout-seconds=2

			livenessProbes:
			  httpGet:
			    path: /healthz
			    port: 8080
			  initial-delay-seconds: 2
			  timeout-seconds: 2
			
			oc set probe dc/<nombre-DC> --liveness --open-tcp=3306 --period=20 --timeout-seconds=1

			livenessProbes:
			  tcpSocket:
			    port: 3306
			  periodSeconds: 20
			  timeout-seconds: 1			
			

Lab 10: Reviews
	
	Review 1:
		Objetivos:
			- Optimizar Dockerfile
			- Adaptar Dockerfile para que pueda correr un openshift
			- Publicar imagen en registro externo
			- Crear una IS en un proyecto compartido.
		
		Troubleshoot:
			-
			- La permisologia de system:image-puller se le da al GRUPO system:accountservices:<NOM-PROJECTO>
			
	Review 3:
		Troubleshoot:
			- Cuando agregas una variable a un secret hay que tener cuidado porque es un recurso que esta codificado. 
			por lo tanto, no se agrega la variable solo colocando el ${VARIABLE}
				- Se debe anteponer a la definicion "Data" "stringData" y se elimina la info codificada y ahora si se puede agregar la variable.
			- El template del nodejs-mongodb-example tiene un ejemplo de probes
		
## DOCUMENTACION:

   1. Dockerfile, permisos:
       - Doc Images, 4.1.2:
	https://access.redhat.com/documentation/en-us/openshift_container_platform/4.5/html-single/images/index#images-create-guide-openshift_create-images
 
   2. Registry:
       - Doc. Registry, 2.4 - Enable Default Route:     
	https://access.redhat.com/documentation/en-us/openshift_container_platform/4.5/html-single/registry/index#registry-operator-default-crd_configuring-registry-operator

   3. S2I: 
      - Doc. Images, 4.3: 
      	https://access.redhat.com/documentation/en-us/openshift_container_platform/4.5/html-single/images/index#images-create-s2i_create-images

   4. WebHook Triggers:
      - Doc. Builds, 8.1.1:
	https://access.redhat.com/documentation/en-us/openshift_container_platform/4.5/html-single/builds/index#builds-webhook-triggers_triggering-builds-build-hooks

   5. Templates:
      - Doc Images, 10.7.8
	https://access.redhat.com/documentation/en-us/openshift_container_platform/4.5/html-single/images/index#templates-create-from-existing-object_using-templates

   6. Health Check
      -Doc Aplicacion Cap. 6
      https://access.redhat.com/documentation/en-us/openshift_container_platform/4.5/html/applications/application-health
