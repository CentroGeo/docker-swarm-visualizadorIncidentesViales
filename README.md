# docker-swarm-visualizadorIncidentesViales
Configuraciones de Docker swarm para el Visualizador de Incidentes Viales


## Instrucciones

````bash
# Configuración de Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

sudo usermod -aG docker ubuntu

# Docker swarm
docker swarm init
# Red pública
docker network create --driver=overlay traefik-public
export NODE_ID=$(docker info -f '{{.Swarm.NodeID}}')

docker node update --label-add traefik-public.traefik-public-certificates=true $NODE_ID
# Dominio (debe estar configurado en un DNS)
export EMAIL=admin@example.com
export DOMAIN=traefik.sys.app.example.com
# Para traefik
export USERNAME=admin
export HASHED_PASSWORD=$(openssl passwd -apr1)

##
# Servicios
##

# 1. Traefik
git clone git@github.com:CentroGeo/docker-swarm-visualizadorIncidentesViales.git
cd docker-swarm-visualizadorIncidentesViales
docker stack deploy -c traefik.yml traefik

# 2. Visualizador (la url debe estar configurada en un DNS)
export APP_DOMAIN=app.example.com
# red interna
docker network create --driver=overlay sp-net
# levantar el visualizador
docker stack deploy -c standaloneapp.yml app
# Replicarlo
docker service update app_visualizador --replicas [No de réplicas deseadas]
docker service update --force app_visualizador
````
