


## Capitulo 4

	Guiado 3: 
		Objetivo: 
			- Crear un commando Post-Commit
		TroubleShoot: 
			- Una de las variables utilizada en el comando post commit no estaba definida como variable de entorno.

		Commando relevante: 

			oc set build-hook bc/name --post-commit --command -- COMANDO
			oc set build-hook bc/name --post-commit --script="SCRIPT"

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
		Troubleshoot: 
			-Inspeccionar la imagen con podman
			-Se debe crear un branch en el codigo para colocar estos ficheros en .s2i/bin
			-Se realizan las modificaciones necesarias a estos ficheros
			-Se crea una nueva app para probar que conje los cambios
			--Se verifican los logs de BC para visualizar los mensajes dejados por los nuevos scripts
			--Se verifican que los cambios hayan sido aplicados.

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
			- Generamos la imagen de construccion en local con podman.
			- Probamos codigo fuente usando la imagen generada utilizando la herramienta s2i.
			- Subimos la imagen de construccion creada y probada a quay.io
			- Creamos un secret con las credenciales de acceso al registry privado
			- Linkeamos el secret con las credenciales a la SA builder para que pueda hacer pulls
			- Importamos el IS de quay.io
			- Probamos el IS con un codigo fuente
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

Lab 10: Reviews
	
	Review 1:
		Objetivos:
		
		Troubleshoot:
			- La permisologia de system:image-puller se le da al GRUPO system:accountservices:<NOM-PROJECTO>
			
	Review 3:
		Troubleshoot:
			- Cuando agregas una variable a un secret hay que tener cuidado porque es un recurso que esta codificado. 
			Por lo tanto, no se agrega la variable solo colocando el ${VARIABLE}
			- El template del nodejs-mongodb-example tiene un ejemplo de probes
		

