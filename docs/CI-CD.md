# Documentation CI/CD - Orion MicroCRM

## 1. Objectif

Orion MicroCRM est une application Full-Stack composée d'un backend Spring Boot et d'un frontend Angular.

L'objectif de la chaîne CI/CD est d'automatiser la construction, les tests, l'analyse de la qualité du code et la publication des images Docker afin de réduire les erreurs liées aux déploiements manuels et de garantir la reproductibilité des livraisons.

La chaîne mise en place s'appuie sur GitHub Actions, Gradle, npm, JaCoCo, Karma, SonarQube Cloud, Docker, Docker Compose et GitHub Container Registry (GHCR).

## 2. Architecture de l'application

Le projet utilise un monorepo contenant le backend Spring Boot, le frontend Angular et les fichiers nécessaires à la chaîne CI/CD.

### 2.1 Backend

Le backend repose sur Spring Boot et Gradle. Java 17 est utilisé dans GitHub Actions et dans l'image Docker. L'application écoute sur le port 8080.

Le fichier `back/Dockerfile` utilise une construction multi-stage : une première image construit l'application avec Gradle, puis le JAR exécutable est copié dans une image Java plus légère.

### 2.2 Frontend

Le frontend repose sur Angular 17. Node.js 20 est utilisé dans la CI.

Le fichier `front/Dockerfile` construit l'application Angular puis copie les fichiers générés dans une image Caddy chargée de servir l'application web.

### 2.3 Docker Compose

Le fichier `docker-compose.yml` permet d'exécuter les deux services ensemble :

- backend : `8080:8080` ;
- frontend HTTP : `8081:80` ;
- frontend HTTPS : `8443:443`.

Le frontend dépend du backend avec `depends_on`.

## 3. Pipeline GitHub Actions

Le pipeline CI/CD est défini dans `.github/workflows/ci.yml`.

Il est déclenché :

- lors d'un `push` sur `main` ;
- lors d'une Pull Request vers `main` ;
- manuellement avec `workflow_dispatch`.

Le pipeline est organisé en deux jobs principaux :

1. `build-test` : construction, tests, couverture et analyse SonarQube Cloud ;
2. `publish-images` : construction et publication des images Docker dans GHCR.

Le job `publish-images` contient `needs: build-test`. Il ne peut donc démarrer que lorsque `build-test` s'est terminé avec succès.

### 3.1 Construction et tests du backend

GitHub Actions utilise Java 17 avec la distribution Temurin.

Le backend est construit et testé avec la commande Linux suivante :

`./gradlew clean build jacocoTestReport copyRuntimeClasspath copyTestRuntimeClasspath --no-daemon`

Cette étape compile l'application, exécute les tests, génère le rapport JaCoCo et prépare les bibliothèques nécessaires à l'analyse SonarQube.

La même procédure a également été vérifiée localement sous Windows avec `gradlew.bat`. Deux tests ont été exécutés avec succès.

### 3.2 Construction et tests du frontend

GitHub Actions utilise Node.js 20 pour le frontend.

Les dépendances sont installées avec `npm ci`, afin d'utiliser les versions définies dans `package-lock.json`.

L'application Angular est ensuite construite avec `npm run build`.

Les tests sont exécutés en mode headless avec :

`npm test -- --watch=false --browsers=ChromeHeadlessNoSandbox --code-coverage`

Lors de la validation locale, les 8 tests Angular ont été exécutés avec succès.

La couverture frontend est exportée au format LCOV dans `front/coverage/microcrm/lcov.info`.

### 3.3 Analyse SonarQube Cloud

Après les constructions et les tests, le pipeline exécute une analyse SonarQube Cloud.

Le projet SonarQube utilise :

- organisation : `abdelbasirmn` ;
- project key : `abdelbasirmn_P7-FSJA-Orion`.

Le secret `SONAR_TOKEN` est stocké dans les secrets GitHub et n'est pas enregistré dans le dépôt.

Les variables `SONAR_ORGANIZATION` et `SONAR_PROJECT_KEY` sont configurées dans les variables GitHub.

SonarQube exploite le rapport JaCoCo du backend et le rapport LCOV du frontend.

Après configuration complète de l'analyse, le Quality Gate a été validé. La mesure observée indiquait une couverture globale de 37,4 % et un taux de duplication global de 2,5 %.

### 3.4 Publication des images Docker dans GHCR

Le second job du pipeline, `publish-images`, dépend de `build-test` grâce à `needs: build-test`.

Il n'est pas exécuté pour les Pull Requests. Les images ne sont donc publiées que lors des exécutions autorisées du workflow, notamment après un push réussi sur `main`.

Le workflow s'authentifie auprès de GitHub Container Registry avec `GITHUB_TOKEN`. Aucun identifiant GHCR n'est stocké directement dans le code source.

Deux images sont construites et publiées :

- `ghcr.io/abdelbasirmn/p7-fsja-orion-back` ;
- `ghcr.io/abdelbasirmn/p7-fsja-orion-front`.

Chaque image reçoit deux tags :

- `latest`, correspondant à la dernière image publiée ;
- le SHA Git complet, permettant d'associer précisément une image au commit qui l'a produite.

La publication a été validée avec le commit `a875766dec72ec488294a0de191d05b1067892ba`.

Les deux packages, backend et frontend, ont été publiés avec succès dans GHCR et sont associés au dépôt `P7-FSJA-Orion`.

## 4. Exécution et vérification

### 4.1 Exécution locale avec Docker Compose

Depuis la racine du dépôt, construire les images et démarrer les services avec :

`docker compose up -d --build`

Vérifier ensuite l'état des conteneurs avec :

`docker compose ps`

Les points d'accès utilisés lors de la validation sont :

- backend : `http://localhost:8080` ;
- frontend HTTP : `http://localhost:8081` ;
- frontend HTTPS : `https://localhost:8443`.

Caddy redirige le trafic HTTP du frontend vers HTTPS. Lors de la validation locale, une requête HTTPS sur le port 8443 a retourné un statut HTTP 200.

### 4.2 Tests du backend sous Windows

Depuis le dossier `back`, les tests et le rapport de couverture peuvent être générés avec :

`.\gradlew.bat clean build jacocoTestReport copyRuntimeClasspath copyTestRuntimeClasspath --no-daemon`

Lors de la dernière vérification locale, les deux tests backend ont réussi et Gradle a retourné `BUILD SUCCESSFUL`.

### 4.3 Tests du frontend

Depuis le dossier `front`, installer les dépendances avec :

`npm ci`

Puis exécuter les tests avec couverture :

`npm test -- --watch=false --browsers=ChromeHeadlessNoSandbox --code-coverage`

La validation effectuée sur le projet a exécuté 8 tests avec succès.
