# Documentation technique --- Orion MicroCRM

**Projet 7 OpenClassrooms --- Mettez en œuvre l'intégration et le
déploiement continu d'une application Full-Stack**\
**Auteur :** MEFIRE NSANGOU Abdelbasir\
**Option choisie :** Option B --- Scénario Orion\
**Date :** 7 octobre 2026

------------------------------------------------------------------------

## 1. Introduction

### 1.1 Contexte du projet

Orion MicroCRM est une application interne de démonstration de type CRM
simplifié. Elle est composée d'un backend **Spring Boot** et d'un
frontend **Angular**.

Dans le scénario du projet, les déploiements manuels génèrent des délais
et augmentent le risque d'erreurs. L'objectif de l'industrialisation est
donc de mettre en place une chaîne CI/CD reproductible permettant
d'automatiser la construction, les tests, l'analyse de qualité, la
conteneurisation et la publication des images de l'application.

Le projet est organisé sous forme de monorepo. Les principaux éléments
sont :

-   `back/` : backend Spring Boot ;
-   `front/` : frontend Angular ;
-   `.github/workflows/ci.yml` : pipeline GitHub Actions ;
-   `docker-compose.yml` : orchestration locale ;
-   `sonar-project.properties` : configuration SonarQube Cloud ;
-   `monitoring/logstash/pipeline/logstash.conf` : pipeline Logstash ;
-   `docs/` : documentation technique détaillée.

### 1.2 Objectifs de l'industrialisation

Les objectifs retenus sont :

-   automatiser la construction du backend et du frontend ;
-   exécuter automatiquement les tests ;
-   générer les rapports de couverture ;
-   analyser la qualité et la sécurité avec SonarQube Cloud ;
-   produire des images Docker reproductibles ;
-   publier les images validées dans GitHub Container Registry (GHCR) ;
-   centraliser les logs du backend avec Logstash, Elasticsearch et
    Kibana ;
-   documenter les KPI observables sans inventer de métriques de
    production ;
-   documenter et tester une stratégie de sauvegarde et de restauration
    des données persistantes de monitoring ;
-   fournir des procédures de maintenance et de retour arrière.

### 1.3 Technologies principales

  -----------------------------------------------------------------------
  Domaine                             Technologie
  ----------------------------------- -----------------------------------
  Backend                             Spring Boot 3.2.5, Java 17, Gradle

  Frontend                            Angular 17, Node.js 20 dans la CI,
                                      npm

  Tests backend                       JUnit / Spring Boot Test

  Tests frontend                      Karma, ChromeHeadlessNoSandbox

  Couverture backend                  JaCoCo

  Couverture frontend                 LCOV

  CI/CD                               GitHub Actions

  Qualité / sécurité                  SonarQube Cloud

  Conteneurisation                    Docker, Docker Compose

  Publication                         GitHub Container Registry

  Serveur frontend                    Caddy

  Monitoring                          Logback JSON, Logstash 8.17.4,
                                      Elasticsearch 8.17.4, Kibana 8.17.4
  -----------------------------------------------------------------------

### 1.4 Vue synthétique du pipeline

La chaîne mise en place suit le principe suivant :

``` text
Push / Pull Request / déclenchement manuel
                  |
                  v
          GitHub Actions
                  |
                  v
     Build + tests backend
                  |
                  v
   Build + tests frontend
                  |
                  v
      Rapports de couverture
                  |
                  v
       SonarQube Cloud
                  |
        succès de build-test
                  |
                  v
  Publication GHCR hors Pull Request
```

La publication d'images dans GHCR constitue une étape de livraison des
artefacts conteneurisés. Elle n'est pas présentée comme un déploiement
en production.

------------------------------------------------------------------------

## 2. Étapes de mise en œuvre du pipeline CI/CD

### 2.1 Structure du pipeline

Le workflow est défini dans `.github/workflows/ci.yml`.

Il est déclenché :

-   lors d'un `push` sur `main` ;
-   lors d'une Pull Request vers `main` ;
-   manuellement avec `workflow_dispatch`.

Aucun déclenchement `schedule` ou nightly n'est actuellement configuré.

Le workflow contient deux jobs principaux.

#### Job `build-test`

L'ordre d'exécution est :

1.  récupération du dépôt avec `actions/checkout@v7` et `fetch-depth: 0`
    ;
2.  configuration de Java 17 avec `actions/setup-java@v6` ;
3.  build et tests du backend ;
4.  génération du rapport JaCoCo ;
5.  préparation des bibliothèques Java nécessaires à SonarQube ;
6.  configuration de Node.js 20 avec `actions/setup-node@v7` ;
7.  installation déterministe des dépendances frontend avec `npm ci` ;
8.  build Angular ;
9.  tests frontend headless et génération de la couverture LCOV ;
10. analyse SonarQube Cloud avec
    `SonarSource/sonarqube-scan-action@v8.3.0`.

#### Job `publish-images`

Le job `publish-images` :

-   dépend de `build-test` avec `needs: build-test` ;
-   n'est pas exécuté pour les Pull Requests ;
-   s'authentifie dans GHCR avec `docker/login-action@v3` et
    `GITHUB_TOKEN` ;
-   construit et publie les images avec `docker/build-push-action@v6`.

Deux images sont publiées :

-   `ghcr.io/abdelbasirmn/p7-fsja-orion-back` ;
-   `ghcr.io/abdelbasirmn/p7-fsja-orion-front`.

Chaque image est publiée avec :

-   le tag `latest` ;
-   un tag correspondant au SHA complet du commit.

Cette stratégie permet de conserver un lien précis entre le code Git et
l'image construite.

#### Justification du choix de GitHub Actions

GitHub Actions a été retenu car le code source est hébergé sur GitHub et
le service permet de placer la définition du pipeline directement dans
le dépôt. Le workflow est donc versionné avec le code, relançable depuis
GitHub et intégré nativement aux événements `push`, `pull_request` et
`workflow_dispatch`.

L'utilisation d'actions spécialisées permet également de standardiser la
préparation des environnements Java et Node, l'authentification GHCR, le
scan SonarQube et la construction des images Docker.

Le workflow ne contient pas actuellement d'étape explicite qui attend le
résultat du Quality Gate SonarQube avant la publication. La publication
dépend du succès du job `build-test`, dont le scan SonarQube fait
partie, mais il ne faut pas assimiler ce comportement à un contrôle
explicite du Quality Gate.

### 2.2 Scripts et commandes d'automatisation

#### Backend

Commande exécutée dans la CI :

``` shell
./gradlew clean build jacocoTestReport copyRuntimeClasspath copyTestRuntimeClasspath --no-daemon
```

Sous Windows :

``` powershell
.\gradlew.bat clean build jacocoTestReport copyRuntimeClasspath copyTestRuntimeClasspath --no-daemon
```

Cette commande :

-   nettoie les sorties précédentes ;
-   compile le backend ;
-   exécute les tests ;
-   génère le rapport JaCoCo ;
-   prépare les dépendances runtime et test utilisées par l'analyse Java
    SonarQube.

#### Frontend

Installation :

``` shell
npm ci
```

Construction :

``` shell
npm run build
```

Tests avec couverture :

``` shell
npm test -- --watch=false --browsers=ChromeHeadlessNoSandbox --code-coverage
```

`npm ci` utilise les versions verrouillées dans `package-lock.json`, ce
qui améliore la reproductibilité de l'installation.

#### Docker Compose

Démarrage complet :

``` shell
docker compose up -d --build
```

Vérification :

``` shell
docker compose ps
```

Arrêt :

``` shell
docker compose down
```

#### Adaptation

Les versions Java et Node sont définies dans le workflow. Les contextes
et Dockerfiles des images sont également déclarés explicitement dans
`ci.yml`. Toute évolution majeure de Java, Node, Spring Boot, Angular ou
des images Docker doit donc être accompagnée d'une mise à jour contrôlée
du workflow, des Dockerfiles et des tests.

### 2.3 Reproductibilité et gestion des secrets

Le pipeline peut être relancé :

-   automatiquement par un nouveau push sur `main` ;
-   automatiquement par une Pull Request vers `main` ;
-   manuellement depuis GitHub grâce à `workflow_dispatch`.

Les informations sensibles ne sont pas enregistrées dans le code source.

Le pipeline utilise notamment :

-   `SONAR_TOKEN` dans les secrets GitHub ;
-   `GITHUB_TOKEN` fourni par GitHub Actions pour l'authentification
    GHCR ;
-   `SONAR_ORGANIZATION` et `SONAR_PROJECT_KEY` dans les variables
    GitHub.

Les secrets ne doivent jamais être copiés dans le README, les fichiers
de configuration versionnés, les captures ou les logs.

------------------------------------------------------------------------

## 3. Plan de conteneurisation et de déploiement

### 3.1 Dockerfile du backend

Le backend utilise un build multi-stage.

Étape de construction :

``` dockerfile
FROM gradle:jdk17 AS build
WORKDIR /src
COPY . .
RUN chmod +x gradlew && ./gradlew clean build --no-daemon
```

Étape runtime :

``` dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /src/build/libs/microcrm-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

Le premier stage contient Gradle et les outils nécessaires à la
compilation. Le second ne conserve que l'environnement Java d'exécution
et le JAR produit. Cette séparation évite d'embarquer l'environnement de
build complet dans l'image runtime.

Construction locale :

``` shell
docker build -f back/Dockerfile -t orion-back:local ./back
```

### 3.2 Dockerfile du frontend

Le frontend utilise également un build multi-stage.

Le premier stage utilise Node.js 20 pour installer les dépendances et
construire Angular. Le second utilise Caddy pour servir les fichiers
statiques générés.

Le contexte de construction doit être la racine du dépôt car le
Dockerfile utilise également `misc/docker/Caddyfile`.

Construction locale :

``` shell
docker build -f front/Dockerfile -t orion-front:local .
```

Le frontend expose les ports 80 et 443 dans le conteneur.

### 3.3 Docker Compose

La stack locale définit cinq services :

  --------------------------------------------------------------------------
  Service                 Rôle                    Accès local
  ----------------------- ----------------------- --------------------------
  `backend`               API Spring Boot         `http://localhost:8080`

  `frontend`              Angular servi par Caddy `http://localhost:8081`,
                                                  `https://localhost:8443`

  `elasticsearch`         stockage des logs       `http://localhost:9200`

  `kibana`                exploration des logs    `http://localhost:5601`

  `logstash`              ingestion JSON TCP      `localhost:5000`
  --------------------------------------------------------------------------

Elasticsearch utilise le volume persistant :

``` text
elasticsearch-data
```

La stack de monitoring est destinée à l'environnement local. La sécurité
Elasticsearch/Kibana est désactivée dans ce contexte et ne doit pas être
reproduite telle quelle en production.

### 3.4 Déploiement et publication

Le projet valide deux modes :

-   exécution locale de l'application avec Docker Compose ;
-   publication automatisée des images backend/frontend dans GHCR.

La publication GHCR fournit des artefacts conteneurisés traçables par
SHA. Le projet ne dispose pas d'un environnement de production cible ;
aucune fréquence de déploiement de production n'est donc revendiquée.

------------------------------------------------------------------------

## 4. Plan de testing périodique

### 4.1 Types de tests automatisés

#### Backend

Le projet contient notamment :

-   un test de chargement du contexte Spring ;
-   un test d'intégration du repository Person.

Lors de la validation, **2 tests backend** ont été exécutés avec succès.

#### Frontend

Les tests Angular sont exécutés avec Karma et Chrome en mode headless.

Lors de la validation, **8 tests frontend** ont été exécutés avec
succès.

#### Qualité et sécurité

SonarQube Cloud analyse le projet après les builds et tests et exploite
:

-   le rapport JaCoCo du backend ;
-   le rapport LCOV du frontend.

Le scan SonarQube complète les tests fonctionnels en fournissant des
informations de qualité, maintenabilité, sécurité, duplication,
couverture et risques de dépendances.

### 4.2 Tests selon l'étape

  ---------------------------------------------------------------------------
  Étape                  Backend       Frontend      SonarQube    Publication
                                                                         GHCR
  --------------- -------------- -------------- -------------- --------------
  Push sur `main`            Oui            Oui            Oui         Oui si
                                                                 `build-test`
                                                                      réussit

  Pull Request               Oui            Oui            Oui            Non
  vers `main`                                                  

  Déclenchement              Oui            Oui            Oui         Oui si
  manuel                                                         `build-test`
                                                                      réussit

  Nightly /        Non configuré  Non configuré  Non configuré  Non configuré
  périodique                                                   

  Release dédiée  Pas de trigger Pas de trigger Pas de trigger Pas de trigger
                      spécifique     spécifique     spécifique     spécifique
  ---------------------------------------------------------------------------

Le nightly build n'est donc **pas implémenté** dans le workflow actuel.
Une évolution possible serait d'ajouter un déclenchement planifié pour
réexécuter périodiquement les tests et analyses de dépendances,
notamment afin de détecter les risques apparaissant sans modification du
code.

Avant une future release de production, il est recommandé d'exiger au
minimum une exécution réussie du pipeline sur le commit exact à livrer
et d'utiliser l'image identifiée par son SHA plutôt que de dépendre
uniquement de `latest`.

### 4.3 Critères de réussite et d'alerte

Les critères actuels sont :

-   les commandes de build doivent terminer sans erreur ;
-   les tests backend doivent réussir ;
-   les tests frontend doivent réussir ;
-   le scan SonarQube doit pouvoir être exécuté correctement ;
-   `publish-images` ne doit démarrer que si `build-test` réussit ;
-   une Pull Request ne doit pas publier d'image GHCR.

Le workflow ne contient pas actuellement de mécanisme explicite bloquant
sur le statut du Quality Gate SonarQube. Ce point constitue une
amélioration possible.

Les risques SonarQube Blocker et High doivent être traités en priorité
dans le plan de remédiation, mais le workflow actuel ne contient pas de
règle automatisée personnalisée bloquant la publication selon leur
nombre.

### 4.4 Objectifs des tests

Les tests poursuivent trois objectifs principaux :

1.  **qualité** : vérifier que les composants peuvent être construits et
    que les tests automatisés passent ;
2.  **non-régression** : détecter les régressions introduites par une
    modification ;
3.  **validation avant livraison** : empêcher la publication des images
    lorsque le job préalable échoue.

------------------------------------------------------------------------

## 5. Plan de sécurité

### 5.1 Résultats SonarQube

Les résultats observés le 7 octobre 2026 indiquaient :

-   **73 issues** sur le code analysé ;
-   **38 issues** dans la vue Top issues utilisée lors de la
    vérification ;
-   **0 Security Hotspot restant à examiner** ;
-   **347 Dependency Risks** ;
-   **978 dépendances analysées** ;
-   **couverture globale : 37,4 %** lors de la mesure retenue ;
-   **duplication globale : 2,5 %** ;
-   **960 lignes de code analysées** lors de la mesure retenue.

Le Quality Gate a été observé en état **Passed** après l'intégration
correcte des rapports de couverture et de la configuration Java.

Les résultats SonarQube constituent un instantané et peuvent évoluer
lors de nouvelles analyses.

#### Risques de dépendances observés

  Sévérité SonarQube      Nombre
  -------------------- ---------
  Blocker                      2
  High                         7
  Medium                     149
  Low                        188
  Info                         1
  **Total**              **347**

Parmi les risques Blocker observés :

-   `org.apache.tomcat.embed:tomcat-embed-core 10.1.20`, associé à
    `CVE-2025-24813` et à un CVSS affiché de 9.8 ;
-   `vite 5.1.7`, associé à `CVE-2025-31125`.

La sévérité SonarQube et le score CVSS sont deux informations
différentes et ne doivent pas être confondus.

#### Code smells et complexité

SonarQube signale des issues de maintenabilité et de fiabilité. Un
exemple observé concerne une opération asynchrone dans le constructeur
de `person-details.component.ts`.

Le projet ne conserve pas dans sa documentation une mesure consolidée
spécifique de complexité cyclomatique permettant de présenter une valeur
fiable. Aucune valeur de complexité n'est donc inventée. Les issues
SonarQube doivent être examinées individuellement pour identifier les
zones nécessitant un refactoring.

### 5.2 Analyse des risques

Les principaux risques identifiés sont :

-   dépendances directes ou transitives vulnérables ;
-   versions obsolètes de dépendances ;
-   secret SonarQube exposé si sa gestion sort des GitHub Secrets ;
-   mauvaise interprétation d'une publication GHCR comme un déploiement
    de production ;
-   utilisation en production de la configuration Elasticsearch locale
    sans sécurité ;
-   utilisation non contrôlée de commandes telles que
    `npm audit fix --force` ;
-   utilisation exclusive du tag `latest`, moins traçable qu'un SHA ;
-   absence actuelle de contrôle explicite du Quality Gate avant
    publication.

### 5.3 Plan d'action et remédiation

#### Actions immédiates

-   analyser les 2 risques Blocker ;
-   identifier les dépendances directes responsables des dépendances
    transitives ;
-   rechercher des versions corrigées compatibles ;
-   reconstruire et retester après chaque mise à jour ;
-   relancer SonarQube Cloud.

#### Actions à court terme

-   traiter les 7 risques High ;
-   planifier les risques Medium ;
-   maintenir les secrets uniquement dans GitHub ;
-   utiliser les tags SHA pour les retours arrière ;
-   envisager un contrôle explicite du Quality Gate dans la CI.

#### Actions à long terme

-   mettre en place une revue périodique des dépendances ;
-   automatiser davantage les contrôles de sécurité ;
-   sécuriser Elasticsearch/Kibana dans tout environnement exposé ;
-   définir une politique de rétention des logs ;
-   mettre en place alertes et tableaux de bord adaptés à un
    environnement de production.

Le détail du plan de remédiation est disponible dans `docs/SECURITE.md`.

------------------------------------------------------------------------

## 6. Monitoring, métriques et KPI

### 6.1 Architecture de monitoring

La chaîne de logs mise en place est :

``` text
Spring Boot
    |
    v
Logback + logstash-logback-encoder
    |
    v
Logstash :5000
    |
    v
Elasticsearch
    |
    v
Kibana
```

Le backend produit des événements JSON contenant notamment le champ :

``` text
service = orion-backend
```

Logstash indexe les événements sous la forme :

``` text
orion-logs-YYYY.MM.dd
```

Un Data View Kibana nommé **Orion Logs** utilise le motif :

``` text
orion-logs-*
```

Dans Discover, la recherche sauvegardée **Orion Backend Logs** permet
d'isoler les événements avec :

``` text
service : "orion-backend"
```

Lors d'une validation, 58 événements backend étaient visibles dans
Kibana. Le test ultérieur de sauvegarde/restauration a vérifié 59
documents. Ces nombres correspondent à des observations réalisées à des
moments différents et ne constituent pas un KPI permanent.

### 6.2 KPI CI/CD observés

Un échantillon réel de 10 exécutions GitHub Actions a été étudié.

  KPI                        Valeur observée
  --------------------- --------------------
  Exécutions                              10
  Succès                                   9
  Échecs                                   1
  Taux de réussite CI                   90 %
  Durée moyenne           221 s / 3 min 41 s
  Durée minimale          109 s / 1 min 49 s
  Durée maximale          392 s / 6 min 32 s

Ces valeurs portent sur l'échantillon observé et ne constituent pas des
objectifs de niveau de service.

Le projet ne dispose pas d'une série de mesures séparant
systématiquement le temps de build, le temps de tests et le temps de
publication pour chaque exécution. La durée globale du workflow est donc
le KPI temporel consolidé retenu. Une amélioration consisterait à
collecter les durées de chaque job ou étape séparément.

Le taux d'erreurs applicatives dans les logs n'a pas été calculé sur une
fenêtre temporelle définie. Kibana permet de filtrer les niveaux de
logs, mais aucune valeur fiable de taux d'erreurs n'est revendiquée. Une
future métrique pourrait être définie comme le nombre d'événements
`ERROR` rapporté au nombre total d'événements sur une période
déterminée.

### 6.3 Métriques DORA

#### Deployment Frequency

Non calculée. La publication GHCR n'est pas assimilée à un déploiement
de production et aucun historique de déploiements de production n'est
disponible.

#### Lead Time for Changes

Non calculé. La durée du workflow ne représente pas le délai complet
entre une modification et son déploiement effectif en production.

#### Change Failure Rate

Non calculé. Le taux de réussite CI de 90 % ne doit pas être assimilé au
Change Failure Rate de production.

#### Mean Time to Restore (MTTR)

Non calculé. Aucun historique d'incidents de production et de
restauration de service n'est disponible.

Le document `docs/DORA-KPI.md` détaille la méthode et les limites de ces
mesures.

### 6.4 Analyse synthétique du monitoring

#### Points forts

-   centralisation réelle des logs backend ;
-   événements JSON structurés ;
-   stockage Elasticsearch persistant ;
-   recherche Kibana validée ;
-   identification du service `orion-backend` ;
-   procédure de sauvegarde/restauration Elasticsearch testée.

#### Tendances observées

Sur les 10 workflows étudiés, 9 ont réussi. Les workflows les plus
récents sont plus longs que les premiers car le pipeline s'est enrichi
avec la couverture, SonarQube et la publication des images Docker.

Il n'existe pas encore suffisamment d'historique de production pour
établir une tendance opérationnelle fiable sur les erreurs applicatives,
les incidents ou les déploiements.

#### Points à améliorer

-   collecter les durées détaillées par étape du pipeline ;
-   mesurer le taux d'erreurs applicatives sur une fenêtre définie ;
-   mettre en place un environnement de production ou de staging pour
    calculer les métriques DORA ;
-   sécuriser la stack ELK dans tout environnement exposé.

#### Dashboards

Le projet dispose d'un Data View `Orion Logs` et d'une recherche
Discover sauvegardée `Orion Backend Logs`.

**Aucun dashboard Kibana dédié n'a été implémenté.** La création d'un
dashboard synthétisant volumes de logs, niveaux `ERROR/WARN`, activité
et tendances constitue une amélioration future.

#### Alertes

**Aucune règle d'alerte Kibana/Elasticsearch n'est actuellement
configurée.**

Dans un environnement de production, des alertes pourraient être
définies sur :

-   hausse anormale des erreurs ;
-   indisponibilité d'un service ;
-   saturation de stockage ;
-   échec de sauvegarde ;
-   dégradation des temps de réponse.

------------------------------------------------------------------------

## 7. Plan de sauvegarde des données

### 7.1 Éléments à sauvegarder

#### Code et configuration

Le code source et les configurations sont versionnés dans Git, notamment
:

-   workflow GitHub Actions ;
-   Dockerfiles ;
-   `docker-compose.yml` ;
-   configuration SonarQube ;
-   pipeline Logstash ;
-   configuration Logback ;
-   documentation.

#### Artefacts de build

Les images backend et frontend publiées dans GHCR sont identifiées par :

-   `latest` ;
-   SHA de commit.

Le SHA constitue la référence recommandée pour retrouver une version
précise.

#### Données applicatives

L'application utilise HSQLDB embarqué. Le projet ne met pas en place une
base applicative externe persistante dans Docker Compose. Il ne faut
donc pas présenter une stratégie de sauvegarde de base applicative de
production comme étant déjà implémentée.

#### Données Elasticsearch

Le service Elasticsearch utilise le volume Docker persistant :

``` text
elasticsearch-data
```

Il contient les données de monitoring qui doivent être protégées.

### 7.2 Procédure de sauvegarde

Une procédure de sauvegarde à froid du volume Elasticsearch a été
documentée et testée dans `docs/SAUVEGARDE-MAINTENANCE.md`.

Le principe est :

1.  arrêter Elasticsearch de manière contrôlée ;
2.  créer un répertoire de sauvegarde ;
3.  archiver le contenu du volume persistant ;
4.  vérifier l'archive ;
5.  redémarrer la stack normale après l'opération.

Cette méthode est adaptée à la démonstration locale du projet. Pour un
environnement de production, les mécanismes de snapshot Elasticsearch
seraient à privilégier selon l'architecture cible.

### 7.3 Procédure de restauration

La restauration a été testée dans un volume indépendant afin de
préserver le volume original.

Le scénario comprend :

1.  création d'un volume temporaire de restauration ;
2.  extraction de l'archive dans ce volume ;
3.  démarrage d'une instance Elasticsearch indépendante ;
4.  vérification de la santé du cluster restauré ;
5.  interrogation des logs Orion restaurés ;
6.  suppression du conteneur et du volume temporaires ;
7.  redémarrage de la stack originale ;
8.  vérification d'Elasticsearch et de Kibana.

Le test documenté a notamment vérifié :

-   29 répertoires d'index dans la sauvegarde ;
-   29 shards primaires actifs dans l'instance restaurée ;
-   aucun shard primaire non assigné ;
-   59 logs du backend Orion récupérables ;
-   conservation du volume original ;
-   retour de Kibana à l'état `available`.

### 7.4 Retour à une version stable

Le rollback applicatif repose sur la traçabilité Git/GHCR :

1.  identifier le dernier commit stable ;
2.  sélectionner l'image Docker portant le SHA correspondant ;
3.  redéployer cette version dans l'environnement cible ;
4.  vérifier le démarrage et les logs ;
5.  documenter l'incident et la correction.

Dans le projet actuel, cette procédure est une stratégie documentée ;
aucun environnement de production n'est revendiqué.

### 7.5 Fréquence et limites

Un plan périodique de contrôle est documenté dans
`docs/SAUVEGARDE-MAINTENANCE.md`.

Les valeurs RPO et RTO ne sont pas inventées. Elles devront être
définies et mesurées selon les exigences d'un futur environnement de
production.

------------------------------------------------------------------------

## 8. Plan de mise à jour

### 8.1 Mise à jour de l'application

#### Backend / Gradle / Spring Boot

Les mises à jour doivent être réalisées progressivement :

1.  identifier les dépendances directes et transitives concernées ;
2.  vérifier la compatibilité avec Java 17 et Spring Boot ;
3.  modifier les versions nécessaires ;
4.  exécuter les tests backend ;
5.  reconstruire l'image Docker ;
6.  relancer SonarQube Cloud ;
7.  comparer les résultats avant/après.

Les risques Tomcat observés doivent notamment être traités via la
dépendance parent appropriée plutôt que par une modification transitive
non maîtrisée.

#### Frontend / npm / Angular

La stratégie est :

1.  analyser les dépendances obsolètes ou vulnérables ;
2.  identifier les mises à jour compatibles ;
3.  mettre à jour de manière contrôlée ;
4.  exécuter `npm ci` sur l'état verrouillé ;
5.  construire Angular ;
6.  exécuter les 8 tests ;
7.  reconstruire l'image frontend ;
8.  relancer SonarQube.

`npm audit fix --force` ne doit pas être appliqué automatiquement sans
analyse car il peut introduire des changements majeurs.

#### Images Docker

Les images de base doivent être réévaluées périodiquement :

-   `gradle:jdk17` ;
-   `eclipse-temurin:17-jre-alpine` ;
-   `node:20-alpine` ;
-   `caddy:2-alpine` ;
-   Elasticsearch 8.17.4 ;
-   Logstash 8.17.4 ;
-   Kibana 8.17.4.

Une montée de version doit être accompagnée d'un build, des tests et
d'une validation de la stack.

### 8.2 Mise à jour du pipeline CI/CD

Le workflow utilise actuellement :

-   `actions/checkout@v7` ;
-   `actions/setup-java@v6` ;
-   `actions/setup-node@v7` ;
-   `SonarSource/sonarqube-scan-action@v8.3.0` ;
-   `docker/login-action@v3` ;
-   `docker/build-push-action@v6`.

Ces versions doivent être revues périodiquement.

La maintenance du pipeline doit comprendre :

1.  lecture des notes de version des actions ;
2.  vérification des changements incompatibles ;
3.  mise à jour d'une action à la fois lorsque cela est pertinent ;
4.  exécution du workflow ;
5.  contrôle des tests et de SonarQube ;
6.  contrôle de la publication GHCR ;
7.  conservation d'un commit facilement réversible.

Les versions Java 17 et Node.js 20 déclarées dans la CI doivent rester
cohérentes avec les versions utilisées par les Dockerfiles et
l'application.

### 8.3 Fréquence et bonnes pratiques

La maintenance recommandée comprend :

-   revue régulière des Dependency Risks SonarQube ;
-   traitement prioritaire des risques Blocker et High ;
-   revue périodique des dépendances npm/Gradle ;
-   revue des images Docker ;
-   revue des versions des GitHub Actions ;
-   exécution des tests après chaque modification ;
-   nouvelle analyse SonarQube après mise à jour ;
-   utilisation des SHA Git et GHCR pour la traçabilité ;
-   sauvegarde Elasticsearch avant une opération de maintenance sensible
    ;
-   vérification de la restauration à intervalles définis.

Un nightly build n'est pas actuellement implémenté. Il peut être ajouté
ultérieurement si le besoin de contrôles périodiques automatisés est
retenu.

------------------------------------------------------------------------

## 9. Conclusion

Le projet Orion MicroCRM dispose désormais d'une chaîne CI/CD versionnée
et reproductible permettant d'automatiser les principales étapes de
validation et de livraison de l'application.

Les améliorations apportées sont notamment :

-   automatisation des builds backend et frontend ;
-   exécution automatique des tests ;
-   génération de la couverture JaCoCo et LCOV ;
-   analyse SonarQube Cloud intégrée ;
-   conteneurisation multi-stage ;
-   publication automatisée d'images backend/frontend dans GHCR ;
-   traçabilité des images par SHA ;
-   centralisation des logs avec Logstash, Elasticsearch et Kibana ;
-   collecte de KPI réels du pipeline ;
-   distinction explicite entre KPI CI et métriques DORA ;
-   analyse des risques de dépendances et plan de remédiation ;
-   sauvegarde et restauration Elasticsearch documentées et testées ;
-   stratégie de maintenance et de rollback.

Les gains principaux sont la **reproductibilité**, la **traçabilité**,
la **réduction des opérations manuelles**, la **détection plus précoce
des erreurs** et une meilleure visibilité sur la qualité et les logs de
l'application.

Le projet conserve néanmoins plusieurs axes d'amélioration :

-   augmenter progressivement la couverture de tests ;
-   traiter les Dependency Risks prioritaires ;
-   ajouter, si nécessaire, un contrôle explicite du Quality Gate avant
    publication ;
-   ajouter des tests périodiques/nightly si le besoin est confirmé ;
-   mettre en place un dashboard Kibana dédié ;
-   définir des alertes ;
-   collecter des KPI par étape du pipeline ;
-   mettre en place un véritable environnement de staging/production
    afin de mesurer les quatre métriques DORA ;
-   sécuriser la stack ELK pour tout environnement exposé.

La solution actuelle constitue donc une base CI/CD complète pour le
périmètre du projet, tout en distinguant clairement les fonctionnalités
réellement implémentées des améliorations prévues pour un environnement
de production.

------------------------------------------------------------------------

## Annexes --- preuves et documents associés

### A. Documentation détaillée

-   `docs/CI-CD.md` : architecture, pipeline, commandes et publication
    GHCR ;
-   `docs/MONITORING.md` : centralisation des logs et validation Kibana
    ;
-   `docs/DORA-KPI.md` : exécutions réelles et positionnement DORA ;
-   `docs/SECURITE.md` : résultats SonarQube et plan de remédiation ;
-   `docs/SAUVEGARDE-MAINTENANCE.md` : sauvegarde, restauration testée,
    rollback et maintenance.

### B. Fichiers techniques principaux

-   `.github/workflows/ci.yml`
-   `back/Dockerfile`
-   `front/Dockerfile`
-   `docker-compose.yml`
-   `sonar-project.properties`
-   `back/src/main/resources/logback-spring.xml`
-   `monitoring/logstash/pipeline/logstash.conf`

### C. Résultats vérifiés pendant le projet

-   backend : 2 tests réussis ;
-   frontend : 8 tests réussis ;
-   SonarQube : Quality Gate observé `Passed` ;
-   couverture globale observée : 37,4 % ;
-   duplication globale observée : 2,5 % ;
-   analyse : 960 lignes de code lors de la mesure retenue ;
-   sécurité : 73 issues, 0 Security Hotspot restant à examiner ;
-   dépendances : 347 risques sur 978 dépendances ;
-   KPI CI : 10 exécutions, 9 succès, 1 échec, 90 % de réussite ;
-   durée moyenne de l'échantillon : 221 secondes ;
-   GHCR : images backend et frontend publiées avec tags `latest` et SHA
    ;
-   Elasticsearch : cluster local validé ;
-   Kibana : Data View `Orion Logs` et recherche `Orion Backend Logs` ;
-   restauration Elasticsearch : procédure testée sur un volume
    indépendant.

### D. Commandes utiles

Démarrer la stack :

``` shell
docker compose up -d --build
```

Vérifier les services :

``` shell
docker compose ps
```

Arrêter la stack :

``` shell
docker compose down
```

Backend sous Windows :

``` powershell
cd back
.\gradlew.bat clean build jacocoTestReport copyRuntimeClasspath copyTestRuntimeClasspath --no-daemon
```

Frontend :

``` shell
cd front
npm ci
npm run build
npm test -- --watch=false --browsers=ChromeHeadlessNoSandbox --code-coverage
```

------------------------------------------------------------------------

**Fin de la documentation technique --- Projet 7 OpenClassrooms ---
Orion MicroCRM**
