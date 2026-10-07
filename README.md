<p align="center">
  <img src="./front/src/favicon.png" width="192px" alt="MicroCRM" />
</p>

# Orion MicroCRM — CI/CD d'une application Full-Stack Java / Angular

MicroCRM est une application de démonstration de type CRM simplifié utilisée dans le cadre du projet 7 du parcours **Développeur Full-Stack Java et Angular**.

L'objectif du projet est d'industrialiser l'intégration, les tests, l'analyse de qualité, la conteneurisation, la publication et le suivi de l'application à l'aide d'une chaîne CI/CD reproductible.

L'application permet principalement la création, la modification et la consultation d'individus associés à des organisations.

![Page d'accueil](./misc/screenshots/screenshot_1.png)
![Édition de la fiche d'un individu](./misc/screenshots/screenshot_2.png)

## Architecture

Le projet est organisé sous la forme d'un monorepo contenant :

- `back/` : API Java avec Spring Boot 3.2.5 ;
- `front/` : application Angular 17 ;
- `monitoring/` : configuration de centralisation des logs avec Logstash ;
- `docs/` : documentation technique du projet ;
- `.github/workflows/ci.yml` : pipeline GitHub Actions ;
- `docker-compose.yml` : orchestration locale des services ;
- `sonar-project.properties` : configuration SonarQube Cloud.

La chaîne de monitoring mise en place suit le flux :

```text
Spring Boot -> Logback JSON -> Logstash -> Elasticsearch -> Kibana
```

## Prérequis

Pour une exécution locale complète avec Docker :

- Git ;
- Docker ;
- Docker Compose.

Pour exécuter les composants directement depuis les sources :

- Java 17 ;
- Node.js 20+ ;
- npm ;
- Google Chrome ou Chromium pour les tests Angular.

La CI utilise **Java 17** et **Node.js 20**.

## Démarrage avec Docker Compose

Depuis la racine du dépôt :

```shell
docker compose up -d --build
```

Vérifier l'état des services :

```shell
docker compose ps
```

Services exposés localement :

| Service | Accès |
|---|---|
| Backend Spring Boot | `http://localhost:8080` |
| Frontend HTTP | `http://localhost:8081` |
| Frontend HTTPS | `https://localhost:8443` |
| Elasticsearch | `http://localhost:9200` |
| Kibana | `http://localhost:5601` |
| Logstash TCP | `localhost:5000` |

Le frontend est servi par Caddy. L'accès HTTP peut être redirigé vers HTTPS. Le certificat HTTPS utilisé localement n'est pas un certificat public de production.

Pour arrêter la stack :

```shell
docker compose down
```

## Backend

### Construction et tests

Sous Windows :

```powershell
cd back
.\gradlew.bat clean build --no-daemon
```

Sous Linux/macOS :

```shell
cd back
chmod +x gradlew
./gradlew clean build --no-daemon
```

Le JAR exécutable est généré dans :

```text
back/build/libs/microcrm-0.0.1-SNAPSHOT.jar
```

### Image Docker du backend

Depuis la racine du dépôt :

```shell
docker build -f back/Dockerfile -t orion-back:local ./back
```

## Frontend

### Installation

```shell
cd front
npm ci
```

### Construction

```shell
npm run build
```

### Tests

```shell
npm test -- --watch=false --browsers=ChromeHeadlessNoSandbox
```

Pour générer la couverture utilisée par SonarQube Cloud :

```shell
npm test -- --watch=false --browsers=ChromeHeadlessNoSandbox --code-coverage
```

### Image Docker du frontend

Le Dockerfile du frontend utilise également la configuration Caddy située dans `misc/docker/`. Le contexte de construction doit donc être la racine du dépôt :

```shell
docker build -f front/Dockerfile -t orion-front:local .
```

## CI/CD avec GitHub Actions

Le workflow `.github/workflows/ci.yml` s'exécute sur les pushes vers `main`, les pull requests vers `main` et peut également être déclenché manuellement avec `workflow_dispatch`.

Le job `build-test` réalise :

1. récupération du dépôt avec l'historique Git nécessaire à l'analyse ;
2. configuration de Java 17 ;
3. construction et tests du backend ;
4. génération du rapport JaCoCo ;
5. préparation des bibliothèques Java nécessaires à SonarQube ;
6. configuration de Node.js 20 ;
7. installation des dépendances frontend ;
8. construction Angular ;
9. tests frontend et génération de la couverture LCOV ;
10. analyse SonarQube Cloud.

Après réussite de `build-test`, et hors pull request, le job `Publish Docker images` construit et publie les images du backend et du frontend dans **GitHub Container Registry (GHCR)**.

Les images sont publiées avec :

- un tag `latest` ;
- un tag correspondant au SHA complet du commit.

## Qualité du code avec SonarQube Cloud

Le projet est analysé par SonarQube Cloud à partir des rapports produits par la CI :

- JaCoCo pour le backend Java ;
- LCOV pour le frontend Angular.

Lors des validations réalisées pendant le projet, le Quality Gate a été obtenu avec succès après intégration des rapports de couverture et correction de la configuration d'analyse Java.

Les mesures SonarQube évoluent à chaque analyse. Les captures et documents du projet doivent donc être considérés comme des mesures prises à un instant donné, et non comme des valeurs permanentes.

## Monitoring et centralisation des logs

La stack locale comprend :

- Elasticsearch 8.17.4 ;
- Logstash 8.17.4 ;
- Kibana 8.17.4.

Le backend utilise Logback avec `logstash-logback-encoder` afin d'envoyer des événements JSON à Logstash sur le port TCP 5000.

Logstash indexe ensuite les événements dans Elasticsearch avec un index de la forme :

```text
orion-logs-YYYY.MM.dd
```

Dans Kibana, une Data View `Orion Logs` utilisant le motif `orion-logs-*` permet d'explorer les événements. Une vue enregistrée `Orion Backend Logs` filtre les événements du service `orion-backend`.

Cette configuration est destinée à l'environnement local du projet. La sécurité Elasticsearch/Kibana est désactivée dans cette stack locale et ne doit pas être reproduite telle quelle en production.

## Sécurité

SonarQube Cloud est également utilisé pour identifier les problèmes de sécurité et les risques liés aux dépendances.

L'analyse réalisée pendant le projet a notamment permis de distinguer :

- les issues de code ;
- les vulnérabilités et risques de dépendances ;
- les niveaux de priorité SonarQube ;
- les Security Hotspots, lorsqu'ils existent.

Les mises à jour de dépendances doivent être réalisées de manière contrôlée avec reconstruction, tests et nouvelle analyse. Les corrections forcées susceptibles d'introduire des changements majeurs, telles qu'un `npm audit fix --force` appliqué sans validation, ne font pas partie de la stratégie retenue.

Le détail et le plan de remédiation sont disponibles dans [`docs/SECURITE.md`](./docs/SECURITE.md).

## KPI CI/CD et métriques DORA

Un échantillon réel de 10 exécutions GitHub Actions a été utilisé pour établir des KPI du pipeline :

- 10 exécutions observées ;
- 9 réussies ;
- 1 échouée ;
- taux de réussite CI observé : 90 % ;
- durée moyenne observée : 221 secondes.

Ces indicateurs sont des **KPI de pipeline**. Ils ne doivent pas être assimilés directement aux quatre métriques DORA.

En particulier, le projet ne dispose pas d'un historique suffisant de déploiements et d'incidents de production pour calculer de manière fiable le Deployment Frequency, le Lead Time for Changes, le Change Failure Rate et le MTTR. Aucune valeur de production n'est donc inventée.

Voir [`docs/DORA-KPI.md`](./docs/DORA-KPI.md).

## Sauvegarde, restauration et maintenance

La stratégie de maintenance documente les éléments à conserver, les procédures de restauration et les principes de mise à jour de la stack.

Les composants applicatifs et les configurations sont versionnés dans Git. Les images publiées dans GHCR sont identifiées par `latest` et par SHA de commit, ce qui améliore la traçabilité des versions construites.

Les procédures détaillées sont disponibles dans [`docs/SAUVEGARDE-MAINTENANCE.md`](./docs/SAUVEGARDE-MAINTENANCE.md).

## Documentation technique

| Document | Contenu |
|---|---|
| [`docs/CI-CD.md`](./docs/CI-CD.md) | Architecture, conteneurisation, pipeline CI/CD et procédures d'exécution |
| [`docs/MONITORING.md`](./docs/MONITORING.md) | Elasticsearch, Logstash, Kibana et centralisation des logs |
| [`docs/DORA-KPI.md`](./docs/DORA-KPI.md) | KPI observés et positionnement par rapport aux métriques DORA |
| [`docs/SECURITE.md`](./docs/SECURITE.md) | Analyse de sécurité et plan de remédiation |
| [`docs/SAUVEGARDE-MAINTENANCE.md`](./docs/SAUVEGARDE-MAINTENANCE.md) | Sauvegarde, restauration et stratégie de maintenance |

## Structure simplifiée du dépôt

```text
P7-FSJA-Orion/
├── .github/
│   └── workflows/
│       └── ci.yml
├── back/
│   ├── Dockerfile
│   └── src/
├── front/
│   ├── Dockerfile
│   └── src/
├── docs/
│   ├── CI-CD.md
│   ├── DORA-KPI.md
│   ├── MONITORING.md
│   ├── SAUVEGARDE-MAINTENANCE.md
│   └── SECURITE.md
├── misc/
├── monitoring/
│   └── logstash/
├── docker-compose.yml
├── README.md
└── sonar-project.properties
```

## Principes retenus

Le projet applique les principes suivants :

- automatiser la construction et les tests ;
- bloquer la publication des images lorsque le job préalable échoue ;
- analyser la qualité et la sécurité du code ;
- produire des rapports de couverture exploitables par SonarQube Cloud ;
- construire des images Docker reproductibles ;
- identifier les images publiées par SHA de commit ;
- centraliser les logs applicatifs ;
- documenter les procédures d'exploitation et de maintenance ;
- ne pas inventer de métriques opérationnelles non observées.

## Projet OpenClassrooms

Ce dépôt correspond au **Projet 7 — Mettez en œuvre l'intégration et le déploiement continu d'une application Full-Stack**, réalisé à partir de l'application MicroCRM fournie comme socle du projet.
