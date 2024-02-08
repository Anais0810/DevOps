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
docker build -t anaisdelcamp/database .
```
Pour lancer un container de la base de données (anaisdelcamp/database) j'utilise la commande suivante : 
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

# TP2 GitHub
### 2-1 What are testcontainers?
Testcontainers est un framework opensource permettant de fournir des instances légères et jetables de bases de données courantes.
### 2-2 Document your Github Actions configurations.
```yml
name: test-backend #nom de la pipline
on:
  push:
    branches: 
      - "**" #les branches sur lesquels la pipline sera lancer

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps: # la liste des étapes
      - name: Checkout code
        uses: actions/checkout@v2.5.0 # récupère le code source

      - name: Set up JDK 17 # mise a jour de java en version 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'adopt'
          
      - name: Build and test with Maven # lancement des test avec maven
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=Anais0810_DevOps -Dsonar.organization=anais0810 -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./tp1/backend/simple-api-student-main/


```
### Document your quality gate configuration.
```bash
mvn -B verify sonar:sonar -Dsonar.projectKey=Anais0810_DevOps -Dsonar.organization=anais0810 -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./tp1/backend/simple-api-student-main/
```
 - -B : execute maven en mode non interactif
 - verify sonar:sonar : vérifie et  lance l'analyse de code source avec SonarCloud
 - -Dsonar.projectKey=Anais0810_DevOps : spécifie clé du projet
 - -Dsonar.organization=anais0810 : spécifie l'organisation
 - -Dsonar.host.url=https://sonarcloud.io : spécifie l'URL de l'instance de SonarCloud où le projet doit être analysé
 - -Dsonar.login=${{ secrets.SONAR_TOKEN }} : spécifie le jeton d'authentification pour se connecter à SonarCloud  stockée dans les secrets GitHub
 - --file ./tp1/backend/simple-api-student-main/ : chemin vers le fichier pom.xml du projet



# TP3 Ansible
### 3-1 Document your inventory and base commands
```yml
all:
 vars:
   ansible_user: centos # le user
   ansible_ssh_private_key_file: /home/anais/cpe/devops/tp3/key/id_rsa # le chemain vers la clé rsa
 children:
   prod:
     hosts: centos@anais.delcamp.takima.cloud #le host
```
```bash
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
```

```bash
ansible all -m service -a "name=httpd state=started" --private-key=<path_to_your_ssh_key> -u centos --become
```
### 3-2 Document your playbook
playbook :
```yml
- hosts: all
  gather_facts: false
  become: true

  roles: docker
    
```
/roles/docker/tasks/main.yml
```yml
- name: Test connection
  ping:
- name: Install device-mapper-persistent-data
  yum:
    name: device-mapper-persistent-data
    state: latest

- name: Install lvm2
  yum:
    name: lvm2
    state: latest

- name: add repo docker
  command:
    cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

- name: Install Docker
  yum:
    name: docker-ce
    state: present

- name: Install python3
  yum:
    name: python3
    state: present

- name: Install docker with Python 3
  pip:
    name: docker
    executable: pip3
  vars:
    ansible_python_interpreter: /usr/bin/python3

- name: Make sure Docker is running
  service: name=docker state=started
  tags: docker
```
### 3-3 Document your docker_container tasks configuration.
Dans app:
```yml
- name: Run app
  docker_container:
    name: backendapistudent
    image: anaisdelcamp/tp1-backapistudent:1.0
    env:
      POSTGRES_USR: "{{POSTGRES_USR}}"
      POSTGRES_DB: "{{POSTGRES_DB}}"
      POSTGRES_URL: "{{POSTGRES_URL}}"
      POSTGRES_PASSWORD: "{{POSTGRES_PASSWORD}}"
    networks:
      - name: app-network
```
Dans database
```yml
- name: Run Database
  docker_container:
    name: database
    image: anaisdelcamp/tp1-database:1.0
    env:
      POSTGRES_USR: "{{POSTGRES_USR}}"
      POSTGRES_DB: "{{POSTGRES_DB}}"
      POSTGRES_PASSWORD: "{{POSTGRES_PASSWORD}}"
    ports:
      - "8090:5432"
    volumes:
      - /data
    networks:
      - name: app-network
```
Dans network
```yml
- name: Create a network
  docker_network:
    name: app-network
```
Dans proxy
```yml
- name: Run HTTPD
  docker_container:
    name: httpserver
    image: anaisdelcamp/tp1-httpserver:1.0
    ports:
      - "80:80"
    networks:
      - name: app-network
```
Dans inventories/settings.yml
```yml
all:
 vars:
   ansible_user: centos
   ansible_ssh_private_key_file: /home/anais/cpe/devops/tp3/key/id_rsa
   POSTGRES_USR: "usr"
   POSTGRES_DB: "db"
   POSTGRES_PASSWORD: "pwd"
   POSTGRES_URL: "database:5432"
 children:
   prod:
     hosts: centos@anais.delcamp.takima.cloud
```
