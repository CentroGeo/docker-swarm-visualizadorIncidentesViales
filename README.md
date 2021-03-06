# docker-swarm-visualizadorIncidentesViales
Configuraciones de Docker Swarm para el Visualizador de Incidentes Viales.

En este repositorio están los archivos de configuración para levantar la aplicación en un cluster de servidores utilizando Docker Swarm. La idea general es configurar un nodo con traefik para servir como balanceador de cargas (manager) y luego tantos nodos como se quiera para replicar la aplicación (workers).

# Prerequisitos

Tenemos que tener un nodo (servidor) configurado con una versión reciente de Docker para servir como balanceador de cargas. Si vamos a correr también aquí procesos de la aplicación entonces tendría que tener unos 4 Gigas de RAM y dos núcleos por lo menos.

Para que el manager (nodo de traefik) se pueda comunicar con los demás necesita tener los siguientes puertos abiertos:

|Protocolo     | Puerto        | Fuente            |
|--------------|---------------|-------------      |
| TCP          | 2377          |managers y workers |
| TCP          | 7946          |managers y workers |
| UDP          | 7946          |managers y workers |
| UDP          | 4789          |managers y workers |
| 50           | Todos         |managers y workers |
| HTTP         | 80            |Cualquiera         |
| HTTPS        | 443           |Cualquiera         |

Para cada nodo (worker) extra que queramos configurar, necesitamos los siguientes puertos:

|Protocolo     | Puerto        | Fuente            |
|--------------|---------------|-------------      |
| TCP          | 7946          |managers y workers |
| UDP          | 7946          |managers y workers |
| UDP          | 4789          |managers y workers |
| 50           | Todos         |managers y workers |

# Instrucciones

### Configurar Docker en cada nodo (manager/worker)
````bash
# Configuración de Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

sudo usermod -aG docker ubuntu
````

### Inicializar Docker swarm (sólo en el manager)
````bash
docker swarm init
````
### Crear red pública
````bash
docker network create --driver=overlay traefik-public
````
### Configurar y desplegar Traefik
Primero configuramos la red del nodo en donde va a correr traefik y le decimos que va a usar un certificado público (para https). Simplemente estamos guardando el id del nodo en una variable de entorno y actualizando el nodo con esa información para que traefik configure el certificado,
````bash
export NODE_ID=$(docker info -f '{{.Swarm.NodeID}}')
docker node update --label-add traefik-public.traefik-public-certificates=true $NODE_ID
````
### Dominio de traefik 
Ahora configuramos el dominio para traefik (en este caso el dominio traefik.sys.app.example.com debe apuntar a la ip pública del nodo). También configuramos un usuario y contraseña (autenticación básica) para el Dashboard de traefik
````bash
export EMAIL=admin@example.com
export DOMAIN=traefik.sys.app.example.com
# Para traefik
export USERNAME=admin
export HASHED_PASSWORD=$(openssl passwd -apr1)
````
### Servicio de traefik
Clonamos este repositorio y utilizamos la configuración que viene en el archivo yml para desplegar el servicio.
````bash
git clone https://github.com/CentroGeo/docker-swarm-visualizadorIncidentesViales.git
cd docker-swarm-visualizadorIncidentesViales
docker stack deploy -c traefik.yml traefik
````

### Visualizador 
Primero configuramos (en una variable de entorno que se va a usar adentro del yml que levanta el servicio) el dominio de la aplicación. Otra vez, app.example.com (la url que vayamos a usar) debe apuntar a la ip pública del nodo manager.
````bash
export APP_DOMAIN=app.example.com
````
Después configuramos la red interna que van a utilizar los nodos para comunicarse entre sí, sin importar si residen en diferentes computadoras físicas.
````bash
docker network create --driver=overlay sp-net
````
Finalmente, levantamos el servicio del visualizador
````bash
docker stack deploy -c standaloneapp.yml app
````
Si visitamos app.example.com (después de un par de minutos), debemos ver el visualizador

### Escalar el servicio
Primero tenemos que unir cada nuevo nodo al Swarm, para eso necesitamos el token del manager que podemos obtener haciendo:
````bash
docker swarm join-token worker
````

Esto nos regresa algo como:
````bash
SWMTKN-1-6crh4a0iu3vr64wmgymvi0zj1v219usp91cwcbtu00cor5avxx-9jmbyac8bvxd2h6crbvwsgm21 172.31.12.19:2377
````
Con lo que podemos ejecutar, en cada nodo que queramos agregar al swarm:
````bash
docker swarm join --token SWMTKN-1-6crh4a0iu3vr64wmgymvi0zj1v219usp91cwcbtu00cor5avxx-9jmbyac8bvxd2h6crbvwsgm21 172.31.12.19:2377
````

Para levantar N nuevos workers:
````bash
docker service update app_visualizador --replicas N
docker service update --force app_visualizador
````

La carga se debería balancear automáticamente entre todos los nodos!!!!

Para que el caché se pueda compartir entre todos los nodos (workers), es necesario que exista una carpeta compartida montada en `/efs/cache` en cada nodo.
