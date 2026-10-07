# Monitoring et centralisation des logs

## 1. Objectif

L'application Orion MicroCRM utilise une stack de monitoring locale basee sur Elasticsearch, Logstash et Kibana afin de centraliser et de consulter les logs du backend Spring Boot.

La chaine de traitement mise en place est la suivante :

Spring Boot -> Logback -> Logstash -> Elasticsearch -> Kibana

## 2. Architecture

### Backend Spring Boot

Le backend utilise logstash-logback-encoder pour produire des evenements JSON structures.

Les logs sont conserves dans la console et envoyes a Logstash par TCP. Ils sont identifies par le champ service avec la valeur orion-backend.

Dans Docker Compose, le backend communique avec Logstash avec les parametres suivants :

- LOGSTASH_HOST : logstash
- LOGSTASH_PORT : 5000

### Logstash

Logstash ecoute les evenements JSON sur le port TCP 5000.

Le pipeline est defini dans :

monitoring/logstash/pipeline/logstash.conf

Les evenements sont envoyes vers Elasticsearch dans des index quotidiens au format :

orion-logs-YYYY.MM.dd

### Elasticsearch

Elasticsearch 8.17.4 stocke les evenements centralises et est accessible localement sur le port 9200.

### Kibana

Kibana 8.17.4 est accessible localement sur le port 5601 et permet de rechercher, filtrer et visualiser les logs stockes dans Elasticsearch.

## 3. Execution de la stack

Depuis la racine du projet, la stack peut etre demarree avec :

docker compose up -d

L'etat des services peut etre verifie avec :

docker compose ps

Les principaux ports utilises sont :

| Service | Port local |
| --- | ---: |
| Backend Spring Boot | 8080 |
| Frontend HTTP | 8081 |
| Frontend HTTPS | 8443 |
| Elasticsearch | 9200 |
| Kibana | 5601 |
| Logstash TCP | 5000 |

## 4. Verification d'Elasticsearch

L'etat du cluster peut etre verifie avec :

curl.exe "http://localhost:9200/_cluster/health?pretty"

Lors de la validation de la stack, Elasticsearch a retourne un etat green.

## 5. Verification de la centralisation des logs

La presence des logs du backend dans Elasticsearch peut etre verifiee avec :

curl.exe "http://localhost:9200/orion-logs-*/_search?pretty&q=service:orion-backend"

Les evenements centralises contiennent notamment les champs :

- @timestamp
- message
- logger_name
- level
- level_value
- service
- thread_name

La validation a confirme la presence d'un evenement provenant du backend conteneurise avec le message :

Started MicroCRMApplication in 17.96 seconds (process running for 20.228)

et avec :

service = orion-backend

## 6. Consultation des logs dans Kibana

Un Data View Kibana nomme Orion Logs a ete cree avec :

- index pattern : orion-logs-*
- champ temporel : @timestamp

Dans Discover, les logs du backend sont isoles avec le filtre KQL :

service : "orion-backend"

La recherche sauvegardee est nommee :

Orion Backend Logs

La vue utilise les colonnes suivantes :

- @timestamp
- level
- message

Lors de la validation, la recherche Kibana affichait 58 evenements correspondant au backend Orion.

Cette vue permet de consulter chronologiquement les evenements applicatifs et notamment de retrouver les messages de demarrage du backend Spring Boot.

## 7. Limites de l'environnement local

La stack Elasticsearch, Logstash et Kibana mise en place est destinee a un environnement local de developpement et de demonstration.

La securite Elasticsearch est desactivee dans Docker Compose. Cette configuration facilite l'execution locale, mais ne doit pas etre utilisee telle quelle en production.

Dans un environnement de production, il faudrait notamment prevoir :

- l'authentification des services ;
- le chiffrement des communications ;
- une gestion securisee des secrets ;
- une politique de retention et de rotation des index ;
- des ressources memoire adaptees ;
- une strategie de sauvegarde et de restauration ;
- des regles d'alerte et de supervision adaptees au niveau de service attendu.
