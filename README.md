TP1 - DOCKER

Database :

- Je crée un fichier Dockerfile dans lequel j'ajoute le code suivant :
FROM postgres:14.1-alpine
EXPOSE 5432
ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd

je récupère l'image alpine et je crée des variables d'environnement pour les infos de de la base de données

- Je crée un reseau pour les différents puissent communiquer directement entre eux avec la commande :     docker network create app-network

- je lance ensuite la création du docker depuis le dossier /TP_DOCKER avec la commande :

$ docker build -t hamza10dridi/bdd_postgresql .
$ docker run --name bdd_postgresql hamza10dridi/bdd_postgresql

- J'ajoute également les sctpis BDD dans un dossier scripts et j'ajoute cette ligne dans le Dockerfile :
COPY scripts/*.sql /docker-entrypoint-initdb.d/
ainsi les scripts seront lancés en BDD

- -v /my/own/datadir:/var/lib/postgresql/data : en ajoutant cela on peut retrouver les données mes si le docker crash

BACKEND API : 

QUESTION 1-2 : Le Dockerfile utilise un build multi-étapes pour optimiser la taille de l'image finale et isoler les dépendances de build. La première étape compile l'application Java avec Maven et la deuxième étape crée une image de runtime minimale en copiant uniquement le JAR compilé. Cela permet d'avoir une image Docker légère et sécurisée, prête à exécuter l'application Spring Boot.

- Pour le backend api on va modifier le fichier application.yml pour la connexion à la bdd en utilisant les variables d'environnement :
 datasource:
    url: jdbc:postgresql://bdd_postgresql:5432/${POSTGRES_DB}
    username: ${POSTGRES_USER}
    password: ${POSTGRES_PASSWORD}
    driver-class-name: org.postgresql.Driver


- On run ensuite le Dockerfile fourni pour lancer le docker

REVERSE PROXY :

- Pour le reverse proxy, on va recupérer le fichier httpd.conf
ServerName localhost

<VirtualHost *:80>
    ProxyPreserveHost On
    ProxyPass / http://simple-api-students:8080/
    ProxyPassReverse / http://simple-api-students:8080/
</VirtualHost>
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so

Cela nous permet d'avoir un point d'entré pour avoir accès à l'API mais aussi à d'autre si l'on souhaite en ajouter mais aussi il sert pour la répartition de charge.

DOCKER COMPOSE :

QUESTION 1-3 :

    docker-compose up: Démarre les services définis dans le fichier docker-compose.yml.
    docker-compose down: Arrête et supprime les conteneurs, les réseaux et les volumes associés aux services définis dans le fichier docker-compose.yml.
    docker-compose build: Construit ou reconstruit les images des services.
    docker-compose ps: Affiche l'état des services.
    docker-compose logs: Affiche les journaux de sortie des services.
    docker-compose exec: Exécute une commande dans un conteneur en cours d'exécution.

    
QUESTION 1-4 :

Voici le docker-compose mis en place 

version: '3.8'

services:
  reverse_proxy:
    build: ./TP1_Devops/TP_HTTP
    ports:
      - "80:80"
    networks:
      - app-network
    container_name: reverse_proxy
    depends_on:
      - simple-api-students

  simple-api-students:
    build: ./TP1_Devops/TP_JAVA/simple-api-student-main
    networks:
      - app-network
    container_name: simple-api-students
    depends_on:
      - bdd_postgresql

  bdd_postgresql:
    build: ./TP1_Devops/TP_Docker
    environment:
      - POSTGRES_DB=db
      - POSTGRES_USER=usr
      - POSTGRES_PASSWORD=pwd
    volumes:
      - /home/hamza/Test:/var/lib/postgresql/data
    networks:
      - app-network
    container_name: bdd_postgresql

networks:
  app-network:

Pour chaque docker on leur indique le dockerfile à utiliser pour créer l'image, le réseau sur lequel ils sont pour pouvoir communiquer, le nom, le port pour reverse proxy pour le point d'entré et enfin les depends_on pour les créer dans le bonne orde.
Ne pas oublier de créer le reseau avant.

PUBLISH :

- On va maintenant publier nos images sur dockerhub pour les réutiliser

QUESTION 1-5 :

- On va simplement de connecter avec docker login
- ensuite tag et push les images
docker tag bdd_postgresql hamza10dridi/bdd_postgresql:1.0
docker push hamza10dridi/bdd_postgresql:1.0  

docker tag simple-api-students hamza10dridi/simple-api-students:1.0
docker push hamza10dridi/simple-api-students:1.0  

docker tag reverse_proxy hamza10dridi/reverse_proxy:1.0
docker push hamza10dridi/reverse_proxy:1.0




TP2 - Github Actions

QUESTION 2-1 :

Testcontainers est une bibliothèque Java qui permet aux développeurs de créer facilement des conteneurs Docker pour leurs tests. Ces conteneurs fournissent des versions temporaires de bases de données et d'autres services nécessaires aux tests d'intégration. Cela garantit que les tests se déroulent dans un environnement similaire à celui de la production, ce qui rend les résultats plus fiables.

- Je push mon projet sur github
- Je crée le fichier main.yml dans le dossier .github/workflows
- Je complète le fichier à l'aide du code fourni pour pouvoir tester avec la commande Maven :

Question 2-2

name: CI devops 2023

on:
  push:
    branches:
      - main
      - develop
  pull_request:

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout GitHub code
        uses: actions/checkout@v2.5.0

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          distribution: 'adopt'
          java-version: '17'

      - name: Build and test with Maven
        run: mvn clean test --file "TP_JAVA/simple-api-student-main/pom.xml"

Lorsque l'on push sur la branche main ou develop

on réalise le job test-backend : on precise la version de l'OS, les info sur le java utilisé et on lance le test du backend avec la commande Maven sur le fichier pom.xml qui contient la stucture et le comportement du projet.


- On complète ensuite le main.yml pour rebuild les images des docker et les pousser sur dockerhub à chaque push.
 
 build-and-push-docker-image:
   needs: test-backend
   # run only when code is compiling and tests are passing
   runs-on: ubuntu-22.04
  
   # steps to perform in job
   steps:
     - name: Checkout code
       uses: actions/checkout@v2.5.0
     - name: set up docker buidx
       uses: docker/setup-buildx-action@v1
     - name: Login to Dockerhub
       uses: docker/login-action@v1
       with:
         username: ${{ secrets.DOCKERHUB_USERNAME }}
         password: ${{ secrets.DOCKERHUB_TOKEN }}
  
     - name: Build image and push backend
       uses: docker/build-push-action@v3
       with:
         # relative path to the place where source code with Dockerfile is located
         context: ./TP_JAVA/simple-api-student-main
         # Note: tags has to be all lower-case
         tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-simple-api:latest
         push: ${{ github.ref == 'refs/heads/main' }}
  
     - name: Build image and push database
         # DO the same for database
       uses: docker/build-push-action@v3
       with:
         # relative path to the place where source code with Dockerfile is located
         context: ./TP_Docker
         # Note: tags has to be all lower-case
         tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-bdd_postgresql:latest
         push: ${{ github.ref == 'refs/heads/main' }}
           
  
     - name: Build image and push httpd
       # DO the same for httpd
       uses: docker/build-push-action@v3
       with:
         # relative path to the place where source code with Dockerfile is located
         context: ./TP_HTTP
         # Note: tags has to be all lower-case
         tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-reverse_proxy:latest
         push: ${{ github.ref == 'refs/heads/main' }}

Tout d'abord on va crée des variables secrets pour stocket les informations de connexions (username, toker) sur dockerhub
On réutilise ces variables d'environnement de la façon suivante : ${{secrets.DOCKERHUB_USERNAME}}

On va donc pouvoir se connecter à dockerhub en utilisant l'action github
Ensuite pour chaque docker on va récupérer le chemin jusqu'au dockerfile et utiliser l'action docker@build-push-action@v3 pour tag les images et les push.


SETUP Quality Gate

- Pour cette partie Sonar je n'ai pas fini de la mettre en place
- J'ai créer un compte récupérer le TOKEN et ajouter dans les variables d'environnement secrets
- Sonarcloud est utilisé pour l'analyse statique du code, cela permet d'automatiser la detection des problèmes de qualité du code. Il nous permet de maintenir un code propre et sécuriser.




TP3 - Ansible


- Je crée un dossier ansible dans mon dossier projet dans lequel je crée un autre dossier inventories, je crée le fichier setup.yml
all:
 vars:
   ansible_user: centos
   ansible_ssh_private_key_file: /home/hamza/TP1_Devops/id_rsa
 children:
   prod:
     hosts: hamza.dridi.takima.cloud

Je renseigne le nom du serveur ainsi que la clé pour me connecter.

- Je teste de ping mon serveur avec la commande : ansible all -i inventories/setup.yml -m ping

Collecte d'informations sur les hôtes (facts) :
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"

Suppression du serveur Apache httpd :
    ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent" --become

Avec Ansible, on peut décrire l'état souhaité de notre serveur et laisser Ansible effectuer automatiquement les mises à jour nécessaires. Chaque fois que j'execute une commande, Ansible garantit que le serveur est configuré selon la description fournie.

DEPLOY YOUR APP :

- J'ai stocker les infos base de données dans un fichier secret.yml que je chiffre avec ansible-vault
- Je crée ensuite les 5 roles en m'inspirant du docker-compose réaliser precedement
---
# tasks file for roles/app
- name: Run Simple API
  docker_container:
    name: simple-api-students
    image: hamza10dridi/tp1_devops-simple-api-students:1.0
    networks:
      - name: app-network

    env:
      POSTGRES_DB: "{{ POSTGRES_DB }}"
      POSTGRES_USER: "{{ POSTGRES_USER }}"
      POSTGRES_PASSWORD: "{{ POSTGRES_PASSWORD }}"

---
# tasks file for roles/database
- name: RUN BDD
  docker_container: 
    name: bdd_postgresql
    image: hamza10dridi/tp1_devops-bdd_postgresql:1.0
    networks:
      - name: app-network

    env:
      POSTGRES_DB: "{{ POSTGRES_DB }}"
      POSTGRES_USER: "{{ POSTGRES_USER }}"
      POSTGRES_PASSWORD: "{{ POSTGRES_PASSWORD }}"
---
# tasks file for roles/docker
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


---
# tasks file for roles/network
- name: Create a network
  docker_network:
    name: app-network
    state: present
---
# tasks file for roles/proxy
- name: Run Proxy
  docker_container:
    name: reverse_proxy
    image: hamza10dridi/tp1_devops-reverse_proxy:1.0
    published_ports:
      - "80:80"
    networks:
      - name: app-network


Je lance ensuite le playbook suivant en précisant l'ordre des roles et en important le fichier secret.yml: 
- hosts: all
  gather_facts: false
  become: true

  vars_files:
    - secret.yml
  roles:
    - docker
    - network
    - database
    - app
    - proxy

J'utiliser la commande suivante : 

ansible-playbook -i inventories/setup.yml --ask-vault-pass playbook.yml
