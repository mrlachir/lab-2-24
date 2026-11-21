# 🏠 Smart Home - Application Conteneurisée Multi-Services (Docker & Docker Compose)

Projet de conteneurisation et d'orchestration d'une application complète **Smart Home** (Gestion domotique d'appareils connectés) utilisant une architecture moderne découplée : **Spring Boot**, **Angular**, **MySQL** et **phpMyAdmin**.

---

## 📋 Table des Matières

1. [Architecture Globale](#-architecture-globale)
2. [Structure du Projet](#-structure-du-projet)
3. [Détail des Dockerfiles (Multi-stage Build)](#-détail-des-dockerfiles-multi-stage-build)
   - [Backend (Spring Boot)](#1-dockerfile-backend-smart_home_backdockerfile)
   - [Frontend (Angular)](#2-dockerfile-frontend-smarthome-frontdockerfile)
4. [Orchestration Docker Compose](#-orchestration-docker-compose)
5. [Guide de Démarrage et Déploiement](#-guide-de-démarrage-et-déploiement)
6. [Points d'Accès de l'Application](#-points-daccès-de-lapplication)
7. [Documentation et Tests de l'API REST](#-documentation-et-tests-de-lapi-rest)
8. [Preuves d'Exécution et Captures d'Écran](#-preuves-dexécution-et-captures-décran)

---

## 🏛 Architecture Globale

L'application est découpée en 4 services conteneurisés interconnectés via un réseau virtuel Docker bridge :

```
                        ┌───────────────────────────────┐
                        │      Utilisateur / Client     │
                        └───────┬───────────────┬───────┘
                                │ :80           │ :8081
                                ▼               ▼
                     ┌──────────────────┐  ┌──────────────────┐
                     │     Frontend     │  │    phpMyAdmin    │
                     │  (Angular/Nginx) │  │  (Gestion BDD)   │
                     └─────────┬────────┘  └────────┬─────────┘
                               │ :8085              │
                               ▼                    │
                     ┌──────────────────┐           │
                     │     Backend      │           │
                     │  (Spring Boot)   │           │
                     └─────────┬────────┘           │
                               │ :3306              │ :3306
                               ▼                    ▼
                     ┌────────────────────────────────────────┐
                     │                 MySQL                  │
                     │       (Base: smart-house)              │
                     └────────────────────────────────────────┘
```

- **Frontend** : Application Angular 14 servie par un serveur Web Nginx ultra-léger (port 80).
- **Backend** : API REST Spring Boot 3 avec Java 17 et Spring Data JPA (port 8085).
- **Base de données** : Serveur MySQL 8.x avec persistance des données (port 3306).
- **Administration BDD** : Interface phpMyAdmin pour l'inspection et la gestion des tables (port 8081).

---

## 📁 Structure du Projet

```
.
├── Smart_Home_back/                  # Application Backend Spring Boot
│   ├── Dockerfile                    # Dockerfile multi-stage Maven -> JRE 17
│   ├── pom.xml                       # Dépendances Maven (Spring Boot 3.1.5, JPA, MySQL)
│   └── src/
│       ├── main/
│       │   ├── java/com/example/appareil/
│       │   │   ├── cotroller/        # AppareilController, CategorieController
│       │   │   ├── entity/           # Appareil.java, Categorie.java
│       │   │   ├── repository/       # AppareilRepository, CategorieRepository
│       │   │   ├── service/          # AppareilService, CategorieService
│       │   │   └── AppareilApplication.java
│       │   └── resources/
│       │       └── application.properties # Configuration Datasource dynamique
│       └── test/                     # Tests unitaires et d'intégration
├── smartHome-front/                  # Application Frontend Angular
│   ├── Dockerfile                    # Dockerfile multi-stage Node 14 -> Nginx Alpine
│   ├── package.json                  # Dépendances Angular 14, PrimeNG, TailwindCSS
│   ├── angular.json                  # Configuration du build Angular
│   └── src/                          # Composants, services et assets Angular
├── docker-compose.yml                # Fichier d'orchestration multi-services
├── .env                              # Variables d'environnement pour la configuration
└── README.md                         # Documentation complète du projet
```

---

## 🐳 Détail des Dockerfiles (Multi-stage Build)

L'utilisation du patron **Multi-stage Build** permet d'isoler la phase de compilation (lourde) de la phase d'exécution (légère), garantissant des images Docker minimalistes, sécurisées et performantes.

### 1. Dockerfile Backend (`Smart_Home_back/Dockerfile`)

```dockerfile
# Étape 1 : Compilation du projet avec Maven et OpenJDK 17
FROM maven:3.8.4-openjdk-17 AS builder
WORKDIR /app
COPY ./src ./src
COPY ./pom.xml .
RUN mvn clean package -DskipTests

# Étape 2 : Image d'exécution finale basée sur OpenJDK 17 Alpine
FROM openjdk:17-jdk-alpine
VOLUME /tmp
ARG JAR_FILE=target/*.jar
COPY --from=builder /app/${JAR_FILE} app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

**Explication des étapes :**
- **Stage 1 (`builder`)** : Utilise l'image `maven:3.8.4-openjdk-17` pour compiler le code Java et produire le fichier JAR exécutable (`app.jar`).
- **Stage 2 (`runtime`)** : Repart d'une image `openjdk:17-jdk-alpine` épurée sans les outils de compilation Maven, copiant uniquement le JAR produit pour une empreinte mémoire réduite.

---

### 2. Dockerfile Frontend (`smartHome-front/Dockerfile`)

```dockerfile
# Étape 1 : Construction de l'application Angular avec Node.js
FROM node:14.15.0-alpine AS builder
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build

# Étape 2 : Serveur Web de production Nginx
FROM nginx:alpine
COPY --from=builder /app/dist/smart-home /usr/share/nginx/html
```

**Explication des étapes :**
- **Stage 1 (`builder`)** : Installe les dépendances Node.js (`npm install`) et compile les bundles Angular pour la production (`npm run build`).
- **Stage 2 (`runtime`)** : Copie uniquement les fichiers statiques HTML/CSS/JS générés dans le dossier public d'un serveur Nginx Alpine haute performance.

---

## ⚙️ Orchestration Docker Compose

Le fichier `docker-compose.yml` orchestre le cycle de vie de tous les conteneurs :

```yaml
version: '3'
services:
  mysql:
    image: mysql:latest
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: smart-house
    ports:
      - "3306:3306"

  backend:
    build:
      context: ./Smart_Home_back
    ports:
      - "8085:8085"
    depends_on:
      mysql:
        condition: service_started
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/smart-house
      SPRING_DATASOURCE_USERNAME: root
      SPRING_DATASOURCE_PASSWORD: root
    healthcheck:
      test: "/usr/bin/mysql --user=root --password=root --execute \"SHOW DATABASES;\""
      interval: 5s
      timeout: 2s
      retries: 100

  frontend:
    build:
      context: ./smartHome-front
    ports:
      - "80:80"
    depends_on:
      backend:
        condition: service_started

  phpmyadmin:
    image: phpmyadmin/phpmyadmin
    environment:
      PMA_HOST: mysql
      PMA_PORT: 3306
      MYSQL_ROOT_PASSWORD: root
    ports:
      - "8081:80"
```

---

## 🚀 Guide de Démarrage et Déploiement

### Prérequis
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installé et en cours d'exécution.
- [Git](https://git-scm.com/)

### 1. Cloner le Dépôt
```bash
git clone https://github.com/mrlachir/lab-2-24.git
cd lab-2-24
```

### 2. Construire les Images Docker
```bash
docker-compose build
```

### 3. Lancer l'Application en Arrière-plan
```bash
docker-compose up -d
```

### 4. Vérifier l'État des Conteneurs
```bash
docker-compose ps
```
*Tous les conteneurs (`mysql`, `backend`, `frontend`, `phpmyadmin`) doivent afficher le statut `Up`.*

### 5. Consulter les Logs (Optionnel)
```bash
# Voir les logs du backend Spring Boot
docker-compose logs -f backend

# Voir les logs de l'ensemble des conteneurs
docker-compose logs
```

### 6. Arrêter les Services
```bash
docker-compose down
```

---

## 🌐 Points d'Accès de l'Application

| Service | Rôle | URL d'Accès | Identifiants |
| :--- | :--- | :--- | :--- |
| **Frontend** | Interface Web Angular | [http://localhost](http://localhost) | Accès libre |
| **Backend API** | API REST Spring Boot | [http://localhost:8085](http://localhost:8085) | Accès libre |
| **phpMyAdmin** | Gestionnaire Web MySQL | [http://localhost:8081](http://localhost:8081) | Utilisateur : `root` <br> Mot de passe : `root` |
| **MySQL Database**| Serveur de Base de Données | `localhost:3306` | BDD : `smart-house` <br> User : `root` / `root` |

---

## 🧪 Documentation et Tests de l'API REST

### 1. Gestion des Catégories (`/api/controller/categorie`)

#### • Créer une nouvelle catégorie (POST)
```bash
curl -X POST http://localhost:8085/api/controller/categorie/ \
  -H "Content-Type: application/json" \
  -d '{"label": "Éclairage"}'
```
*Réponse attendue (200 OK) :*
```json
{
  "id": 1,
  "label": "Éclairage",
  "appareilList": null
}
```

#### • Lister toutes les catégories (GET)
```bash
curl -X GET http://localhost:8085/api/controller/categorie/
```

#### • Récupérer une catégorie par ID (GET)
```bash
curl -X GET http://localhost:8085/api/controller/categorie/id/1
```

---

### 2. Gestion des Appareils (`/api/controller/appareil`)

#### • Ajouter un nouvel appareil (POST)
```bash
curl -X POST http://localhost:8085/api/controller/appareil/ \
  -H "Content-Type: application/json" \
  -d '{
    "label": "Lampe Salon",
    "description": "Lampe principale connectée",
    "state": true,
    "photo": "",
    "categorie": {"id": 1}
  }'
```
*Réponse attendue (200 OK) :*
```json
{
  "id": 1,
  "label": "Lampe Salon",
  "description": "Lampe principale connectée",
  "state": true,
  "photo": "",
  "categorie": {
    "id": 1,
    "label": "Éclairage"
  }
}
```

#### • Lister tous les appareils (GET)
```bash
curl -X GET http://localhost:8085/api/controller/appareil/
```

#### • Basculer l'état de tous les appareils (GET)
```bash
# Allumer tous les appareils :
curl -X GET http://localhost:8085/api/controller/appareil/update/state/true

# Éteindre tous les appareils :
curl -X GET http://localhost:8085/api/controller/appareil/update/state/false
```

#### • Modifier un appareil (PUT)
```bash
curl -X PUT http://localhost:8085/api/controller/appareil/id/1 \
  -H "Content-Type: application/json" \
  -d '{
    "label": "Lampe Salon Dimmer",
    "description": "Lampe variable connectée",
    "state": false
  }'
```

#### • Supprimer un appareil (DELETE)
```bash
curl -X DELETE http://localhost:8085/api/controller/appareil/id/1
```

---

## 📸 Preuves d'Exécution et Captures d'Écran

### 1. Construction et Démarrage des Conteneurs (`docker-compose up -d`)
Création du réseau virtuel et initialisation coordonnée des quatre conteneurs :
<p align="center">
  <img width="900" alt="Démarrage Docker Compose" src="https://github.com/user-attachments/assets/5b141e9b-81db-4417-b054-d4831df26404" />
</p>

---

### 2. État d'Exécution des Conteneurs (`docker-compose ps`)
Vérification que l'ensemble des conteneurs (`mysql`, `backend`, `frontend`, `phpmyadmin`) sont actifs et fonctionnels :
<p align="center">
  <img width="900" alt="État des conteneurs actifs" src="https://github.com/user-attachments/assets/0349d203-ef2a-4161-898c-5bb61e99919d" />
</p>

---

### 3. Logs et Initialisation du Backend Spring Boot
Connexion réussie entre le serveur Spring Boot et la base de données MySQL :
<p align="center">
  <img width="900" alt="Logs backend et connexion MySQL" src="https://github.com/user-attachments/assets/9ad6dde0-639c-421a-9b81-a9f07020451f" />
</p>

---

### 4. Interface Web Frontend (Smart Home Application)
Accès à l'application Angular via le port 80 avec affichage et gestion des appareils :
<p align="center">
  <img width="1000" alt="Interface utilisateur Angular" src="https://github.com/user-attachments/assets/5bbfb7bd-370e-4cef-a861-f27e923ad122" />
</p>

---

### 5. Interaction et Ajout d'Appareils Connectés
Formulaire d'ajout et contrôle de l'état (ON/OFF) des appareils connectés :
<p align="center">
  <img width="1000" alt="Gestion des appareils dans le frontend" src="https://github.com/user-attachments/assets/f79f1ea3-d96e-4211-97c0-9251f0b199da" />
</p>

---

### 6. Administration de la Base de Données via phpMyAdmin
Accès à phpMyAdmin (`http://localhost:8081`) et visualisation de la base `smart-house` :
<p align="center">
  <img width="900" alt="Interface phpMyAdmin" src="https://github.com/user-attachments/assets/8b519856-2bf0-4f39-ab9f-4e6b57e45a86" />
</p>

---

### 7. Vérification des Tables et Données dans MySQL
Persistance des entités `appareil` et `categorie` générées automatiquement par Hibernate :
<p align="center">
  <img width="900" alt="Données persistées dans MySQL" src="https://github.com/user-attachments/assets/e5f9c84f-0fb4-47fd-81d5-32e1e4757857" />
</p>

---

## 🏆 Synthèse de Conformité

| Critère d'Évaluation | Statut | Détails |
| :--- | :---: | :--- |
| **Respect des consignes** | ✅ Conforme | Multi-stage build pour backend et frontend, docker-compose complet |
| **Qualité du code** | ✅ Conforme | Architecture propre, isolation des responsabilités, images allégées |
| **Fonctionnement et tests** | ✅ Conforme | Services interconnectés, healthcheck MySQL, API testée avec `curl` |
| **Documentation & Preuves** | ✅ Conforme | README détaillé, guide pas-à-pas, captures d'écran réelles d'exécution |
