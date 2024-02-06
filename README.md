# DevOpsCourse
Rendu des TP du cours DevOps


# TP1 - Docker
### 1-1 Document your database container essentials: commands and Dockerfile.

Pour instancier l'image de la base de données, j'utilise le Dockerfile suivant : 
``` Dockerfile
FROM postgres:14.1-alpine #choisir une image

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr #déféinir variable environnement

COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d
COPY 02-InsertData.sql /docker-entrypoint-initdb.d #copier les scripted dans le docker
```
Avec la commande : 
```
docker build -t nathanglmt/myfirstpostgres .
```
Pour lancer un container de la base de données (nathanglmt/myfirstpostgres) j'utilise la commande suivante : 
``` bash
docker run \ 
-p 8888:5432 \ #définir les pors
--name database \ # nom du containeur
-e POSTGRES_PASSWORD=pwd \ # variable d'environnement
--network app-network \ # choix du réseau
-v data:/var/lib/postgresql/data \ # défoinir le volume
-d \ # pouvoir avoir accès au terminal
anaisdelcamp/database # nom de l'image
```
### 1-2 Why do we need a multistage build? And explain each step of this dockerfile.

``` Dockerfile
# Build
#image de base
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
#variable d'environnement
ENV MYAPP_HOME /opt/myapp
#dossier de travail
WORKDIR $MYAPP_HOME
#copie le pom.xml dans /opt/myapp
COPY pom.xml .
#copie le dossier src dans /opt/myapp
COPY src ./src
#génere les package sans test
RUN mvn package -DskipTests

# Run
#image de base
FROM amazoncorretto:17
#variable d'environnement
ENV MYAPP_HOME /opt/myapp
#dossier de travail
WORKDIR $MYAPP_HOME
#ca récupère le jar géné"rer avant pour le copier dans le nouveau containeur
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
# execute le jar
ENTRYPOINT java -jar myapp.jar
```

### 1-3 Document docker-compose most important commands. 
 - docker compose up : lance le docker
 - docker compose down : arrete le docker
 - docker compose logs : affiche les logs

### 1-4 Document your docker-compose file.
```yml
version: "3.7"

services:
   #pour le back
  backend:
    # on choisit l'image
    image: anaisdelcamp/tp1-backapistudent:1.0
    #on défini les variable d'environnement
    environment:
      - POSTGRES_USR=usr
      - POSTGRES_DB=db
      - POSTGRES_URL=database:5432
      - POSTGRES_PASSWORD=pwd
   # on défénie le réseaux
    networks:
      - app-network
   # on dit de quoi depend ce container
    depends_on:
      - database
   # nom du container
    container_name: backendapistudent

  database:
    image: anaisdelcamp/tp1-database:1.0
    # build:
    #   context: ./database
    environment:
      - POSTGRES_USR=usr
      - POSTGRES_DB=db
      - POSTGRES_PASSWORD=pwd
   # on définie les port
    port:
      - "8888:5432"
    networks:
      - app-network
   # défini le volume pour sauvegarder les donnée a l'arret du container
    volumes:
      - dataDir:/var/lib/postgresql/data
    container_name: database

  httpd:
    image: anaisdelcamp/tp1-httpserver:1.0
    ports:
      - "5050:80"
    networks:
      - app-network
    depends_on:
      - backend
    container_name: httpserver

  adminer:
    image: adminer
    restart: always
    networks:
      - app-network
    ports:
      - 8090:8080

networks:
  app-network:

volumes:
  dataDir:
```

### 1-5 Document your publication commands and published images in dockerhub.

```bash
docker tag anaisdelcamp/database anaisdelcamp/tp1-database:1.0
docker tag anaisdelcamp/backapistudent anaisdelcamp/tp1-backapistudent:1.0
docker tag anaisdelcamp/httpserver anaisdelcamp/tp1-httpserver:1.0
```
Ces commandes enregistent les images (anaisdelcamp/database par exemple) sous un autre nom suivit du numero de la version de l'image (anaisdelcamp/tp1-database:1.0)

```bash
docker push anaisdelcamp/tp1-database:1.0
docker push anaisdelcamp/tp1-backapistudent:1.0
docker push anaisdelcamp/tp1-httpserver:1.0
```
Ces commandes push suer le site de docker les images précédament enregistrer