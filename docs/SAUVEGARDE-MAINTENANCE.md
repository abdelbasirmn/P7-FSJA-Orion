# Sauvegarde, restauration et stratégie de maintenance

## 1. Objectif

Ce document décrit la stratégie de sauvegarde, de restauration et de maintenance mise en place pour le projet Orion MicroCRM.

L'objectif est de :

- protéger les données persistantes de la stack de monitoring ;
- vérifier qu'une sauvegarde peut réellement être restaurée ;
- définir une procédure de mise à jour contrôlée des dépendances ;
- prévoir une stratégie de retour arrière en cas d'échec d'un déploiement.

Les procédures de sauvegarde et de restauration décrites dans ce document ont été testées dans l'environnement Docker local du projet.

---

## 2. Périmètre des données persistantes

### 2.1 Application Orion MicroCRM

Dans l'état actuel du projet, le backend Spring Boot utilise HSQLDB.

Aucune base de données applicative persistante de production n'est configurée dans le fichier `docker-compose.yml`.

Il serait donc incorrect de présenter une procédure de sauvegarde PostgreSQL, MySQL ou d'une autre base de données comme étant actuellement mise en œuvre dans Orion.

Dans un environnement de production, une base de données persistante devra disposer de sa propre stratégie de sauvegarde, indépendante de celle du monitoring.

### 2.2 Elasticsearch

La stack de monitoring utilise un volume Docker persistant :

```text
elasticsearch-data
```

Dans l'environnement Docker Compose local testé, le nom concret du volume est :

```text
p7-fsja-main_elasticsearch-data
```

Ce volume contient notamment les index Elasticsearch dans lesquels Logstash centralise les logs de l'application.

---

## 3. Sauvegarde Elasticsearch testée

### 3.1 Principe

Le test réalisé utilise une sauvegarde à froid du volume Docker Elasticsearch.

Elasticsearch est arrêté avant la copie afin d'éviter de sauvegarder le contenu du volume pendant des opérations d'écriture.

Cette procédure est adaptée à la démonstration et à l'environnement local du projet.

Pour une infrastructure de production, le mécanisme Snapshot/Restore natif d'Elasticsearch avec un repository de snapshots serait préférable à une copie directe du volume Docker.

### 3.2 Arrêt d'Elasticsearch

La commande utilisée est :

```powershell
docker compose stop elasticsearch
```

### 3.3 Répertoire de sauvegarde

Un répertoire de sauvegarde situé en dehors du dépôt Git est utilisé :

```text
C:\Users\User\orion-backups
```

Les archives contenant les données Elasticsearch ne doivent pas être ajoutées au dépôt Git.

### 3.4 Création de l'archive

Une première tentative avec l'image Alpine a rencontré un problème réseau lors du téléchargement de l'image.

Une image `postgres:13` déjà disponible localement et contenant GNU tar a donc été utilisée.

L'option `--pull=never` garantit qu'aucun téléchargement d'image n'est nécessaire pendant l'opération.

La commande utilisée est :

```powershell
docker run --rm --pull=never `
  -v p7-fsja-main_elasticsearch-data:/source:ro `
  -v "${HOME}\orion-backups:/backup" `
  postgres:13 `
  tar -czf /backup/elasticsearch-data-2026-10-07.tar.gz -C /source .
```

Le volume Elasticsearch est monté en lecture seule avec `:ro`.

L'archive obtenue est :

```text
elasticsearch-data-2026-10-07.tar.gz
```

La taille observée pendant le test est :

```text
2 360 863 octets
```

### 3.5 Vérification de l'archive

Le contenu de l'archive a été contrôlé avant toute restauration :

```powershell
docker run --rm --pull=never `
  -v "${HOME}\orion-backups:/backup:ro" `
  postgres:13 `
  tar -tzf /backup/elasticsearch-data-2026-10-07.tar.gz
```

La commande a permis de retrouver les données Elasticsearch, notamment les répertoires d'index, les segments Lucene, les translogs et les métadonnées du nœud.

La sauvegarde n'est toutefois pas considérée comme validée uniquement parce que l'archive peut être ouverte. Un test réel de restauration a également été effectué.

---

## 4. Test réel de restauration

Afin de protéger les données originales, la restauration a été effectuée dans un nouveau volume Docker temporaire.

Le volume original n'a pas été écrasé.

### 4.1 Création du volume temporaire

```powershell
docker volume create orion-elasticsearch-restore-test
```

### 4.2 Restauration de l'archive

```powershell
docker run --rm --pull=never `
  -v orion-elasticsearch-restore-test:/restore `
  -v "${HOME}\orion-backups:/backup:ro" `
  postgres:13 `
  tar -xzf /backup/elasticsearch-data-2026-10-07.tar.gz -C /restore
```

Après extraction, le contenu du volume restauré a été contrôlé.

Le test a permis d'observer :

```text
29 répertoires d'index Elasticsearch
```

ainsi que les fichiers et répertoires nécessaires au fonctionnement du nœud Elasticsearch.

### 4.3 Démarrage d'Elasticsearch sur le volume restauré

Une instance Elasticsearch indépendante a ensuite été démarrée avec le volume restauré.

La même version que celle utilisée par le projet, `8.17.4`, a été conservée.

Le port `9201` a été utilisé afin de ne pas entrer en conflit avec l'instance normale utilisant le port `9200`.

```powershell
docker run -d --name orion-es-restore-test `
  --pull=never `
  -p 9201:9200 `
  -e "discovery.type=single-node" `
  -e "xpack.security.enabled=false" `
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" `
  -v orion-elasticsearch-restore-test:/usr/share/elasticsearch/data `
  docker.elastic.co/elasticsearch/elasticsearch:8.17.4
```

---

## 5. Validation de la restauration

### 5.1 Santé du cluster restauré

La santé de l'instance restaurée a été contrôlée avec :

```powershell
curl.exe "http://localhost:9201/_cluster/health?pretty"
```

Les résultats observés étaient :

- statut : `yellow` ;
- `timed_out` : `false` ;
- nombre de nœuds : 1 ;
- nombre de nœuds de données : 1 ;
- shards primaires actifs : 29 ;
- shards actifs : 29 ;
- shards en cours de déplacement : 0 ;
- shards en initialisation : 0 ;
- shards non assignés : 1 ;
- shards primaires non assignés : 0 ;
- tâches en attente : 0.

Le statut `yellow` ne signifie donc pas que les données sont indisponibles.

Tous les shards primaires nécessaires sont actifs et aucun shard primaire n'est non assigné.

### 5.2 Vérification des logs Orion restaurés

Une requête a ensuite été exécutée directement sur l'instance restaurée :

```powershell
curl.exe "http://localhost:9201/orion-logs-*/_search?pretty&q=service:orion-backend&size=3&sort=@timestamp:desc"
```

La recherche a retrouvé :

```text
59 documents correspondant au service orion-backend
```

La requête Elasticsearch a été exécutée avec succès :

```text
total shards : 1
successful : 1
failed : 0
```

Parmi les événements restaurés figuraient notamment :

```text
Started MicroCRMApplication in 17.96 seconds (process running for 20.228)
```

ainsi que :

```text
Applied InitialDataFixture fixture of set DICTIONARY in 0.110 seconds
```

et :

```text
Tomcat started on port 8080 (http) with context path ''
```

Les événements contenaient également les champs structurés attendus, notamment :

- `@timestamp` ;
- `logger_name` ;
- `level` ;
- `service` ;
- `thread_name`.

La valeur :

```text
service = orion-backend
```

était correctement conservée.

Le test démontre donc que les données sauvegardées peuvent être restaurées dans une nouvelle instance Elasticsearch puis interrogées normalement.

---

## 6. Nettoyage de l'environnement de restauration

Une fois la restauration validée, l'environnement temporaire a été supprimé.

### 6.1 Suppression du conteneur de test

```powershell
docker rm -f orion-es-restore-test
```

### 6.2 Suppression du volume temporaire

```powershell
docker volume rm orion-elasticsearch-restore-test
```

Après suppression, la commande :

```powershell
docker volume ls
```

a confirmé que le volume temporaire n'était plus présent tandis que le volume original était toujours disponible :

```text
p7-fsja-main_elasticsearch-data
```

Cette méthode permet de tester une restauration sans écraser les données originales.

---

## 7. Retour au fonctionnement normal

Après le test de restauration, la stack originale a été redémarrée avec :

```powershell
docker compose up -d
```

Les cinq services du projet ont été retrouvés en fonctionnement :

- backend ;
- frontend ;
- Elasticsearch ;
- Kibana ;
- Logstash.

Les ports utilisés sont notamment :

| Service | Port |
| --- | ---: |
| Backend | 8080 |
| Frontend HTTP | 8081 |
| Frontend HTTPS | 8443 |
| Elasticsearch | 9200 |
| Logstash | 5000 |
| Kibana | 5601 |

### 7.1 Récupération des données Elasticsearch originales

Les logs de démarrage Elasticsearch ont confirmé :

```text
recovered [29] indices into cluster_state
```

Le nœud Elasticsearch a ensuite atteint l'état `started`.

La santé du cluster original a été vérifiée avec :

```powershell
curl.exe "http://localhost:9200/_cluster/health?pretty"
```

Résultat observé :

- statut : `yellow` ;
- `timed_out` : `false` ;
- 1 nœud ;
- 1 nœud de données ;
- 29 shards primaires actifs ;
- 29 shards actifs ;
- 1 shard non assigné ;
- 0 shard primaire non assigné ;
- 0 tâche en attente.

### 7.2 Vérification des logs après redémarrage

La commande suivante a été exécutée :

```powershell
curl.exe "http://localhost:9200/orion-logs-*/_count?q=service:orion-backend"
```

Résultat :

```json
{
  "count": 59,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  }
}
```

Les 59 documents étaient donc toujours présents dans l'environnement original après le test de sauvegarde et de restauration.

### 7.3 Vérification de Kibana

L'état de Kibana a été contrôlé avec :

```powershell
curl.exe -s http://localhost:5601/api/status
```

Le statut global retourné était :

```text
available
```

avec le résumé :

```text
All services and plugins are available
```

Le composant Elasticsearch était également indiqué comme :

```text
available
```

Le retour au fonctionnement normal de la stack de monitoring est donc validé.

---

## 8. Stratégie de maintenance des dépendances

Les dépendances applicatives ne doivent pas être mises à jour automatiquement sans validation.

La stratégie proposée est la suivante :

1. analyser régulièrement les Dependency Risks dans SonarQube Cloud ;
2. traiter en priorité les risques `Blocker`, puis `High` ;
3. identifier si la dépendance concernée est directe ou transitive ;
4. vérifier les notes de version et les changements incompatibles ;
5. effectuer la mise à jour dans une branche dédiée ;
6. reconstruire le backend et le frontend ;
7. exécuter les tests automatisés ;
8. générer les rapports de couverture ;
9. exécuter l'analyse SonarQube Cloud ;
10. vérifier le Quality Gate ;
11. reconstruire les images Docker ;
12. tester l'application avec Docker Compose ;
13. fusionner uniquement après validation.

### 8.1 État observé dans SonarQube Cloud

Lors de l'analyse réalisée dans le cadre du projet, SonarQube Cloud indiquait :

```text
347 Dependency Risks
978 dépendances analysées
```

Répartition observée :

| Sévérité | Nombre |
| --- | ---: |
| Blocker | 2 |
| High | 7 |
| Medium | 149 |
| Low | 188 |
| Info | 1 |
| **Total** | **347** |

Les risques `Blocker` observés incluaient notamment :

- `CVE-2025-24813` concernant `org.apache.tomcat.embed:tomcat-embed-core` 10.1.20 ;
- `CVE-2025-31125` concernant `vite` 5.1.7.

Ces dépendances doivent être traitées de manière contrôlée en conservant les tests, l'analyse SonarQube et la validation fonctionnelle dans le processus.

### 8.2 Mise à jour npm

La commande suivante n'est pas utilisée automatiquement :

```text
npm audit fix --force
```

Une mise à jour forcée peut introduire des versions majeures ou des incompatibilités sans validation préalable.

Les mises à jour doivent être effectuées progressivement puis validées par la CI.

---

## 9. Stratégie de retour arrière

Les images Docker du projet sont publiées dans GitHub Container Registry.

Chaque publication utilise :

- un tag `latest` ;
- un tag correspondant au SHA complet du commit Git.

Exemple de principe :

```text
ghcr.io/<owner>/p7-fsja-orion-back:<commit-sha>
ghcr.io/<owner>/p7-fsja-orion-front:<commit-sha>
```

Le SHA permet d'identifier une image construite à partir d'une version précise du dépôt.

Contrairement à `latest`, ce tag permet de revenir explicitement vers une version antérieure connue.

En cas de régression après une mise à jour :

1. identifier le dernier commit validé ;
2. identifier les images Docker associées à son SHA ;
3. redéployer ces images ;
4. vérifier le backend ;
5. vérifier le frontend ;
6. vérifier Elasticsearch, Logstash et Kibana ;
7. analyser la cause de la régression avant une nouvelle publication.

Le dépôt Git et les tags SHA des images constituent ainsi la base de la stratégie de rollback applicatif.

---

## 10. Stratégie de sauvegarde proposée

Pour l'environnement actuel, les sauvegardes concernent principalement les données persistantes Elasticsearch.

Une stratégie périodique peut être organisée autour des principes suivants :

- sauvegarder avant une opération de maintenance importante ;
- stocker les sauvegardes hors du dépôt Git ;
- protéger les sauvegardes contre les modifications accidentelles ;
- conserver plusieurs générations de sauvegardes ;
- vérifier régulièrement la lisibilité des archives ;
- réaliser périodiquement une restauration dans un environnement isolé ;
- ne jamais considérer une sauvegarde comme fiable uniquement parce que le fichier existe.

Dans une infrastructure de production Elasticsearch, le mécanisme Snapshot/Restore natif devra être privilégié.

---

## 11. Plan périodique proposé

Le planning suivant constitue une recommandation de maintenance.

Il ne représente pas un historique d'opérations déjà exécutées.

| Fréquence proposée | Contrôle |
| --- | --- |
| À chaque push / pull request | Build, tests et analyse SonarQube Cloud |
| Hebdomadaire | Revue des nouveaux problèmes de sécurité et Dependency Risks |
| Mensuelle | Revue des dépendances et mises à jour contrôlées |
| Mensuelle | Test d'une restauration de sauvegarde |
| Avant une mise à jour importante | Création et vérification d'une sauvegarde |
| Après une mise à jour | Tests applicatifs, SonarQube, logs et monitoring |
| Après une modification CI/CD | Vérification du pipeline GitHub Actions |
| Après un incident | Analyse de la cause et validation de la restauration ou du rollback |

La fréquence devra être adaptée aux exigences réelles de l'environnement de production.

---

## 12. RPO et RTO

Aucun RPO ou RTO contractuel n'a été défini ou mesuré dans le cadre du projet.

Il serait donc incorrect de présenter une valeur arbitraire comme un engagement réel.

### 12.1 RPO

Le Recovery Point Objective représente la quantité maximale de données que l'organisation accepte de perdre après un incident.

Il dépend notamment :

- de la fréquence des sauvegardes ;
- de la fréquence d'écriture des données ;
- de la criticité des informations.

### 12.2 RTO

Le Recovery Time Objective représente le temps maximal attendu pour restaurer un service après un incident.

Il dépend notamment :

- du volume de données ;
- du temps de restauration ;
- du temps de redémarrage des services ;
- des contrôles fonctionnels nécessaires.

Les exercices périodiques de restauration permettront de mesurer des durées réelles et d'établir ultérieurement des objectifs adaptés aux besoins métier d'Orion.

---

## 13. Sécurité des sauvegardes

Dans l'environnement local de démonstration, Elasticsearch est configuré avec :

```text
xpack.security.enabled=false
```

Cette configuration simplifie les échanges locaux entre Elasticsearch, Logstash et Kibana.

Elle n'est pas destinée à une exposition en production.

Pour une infrastructure de production, il faudra notamment prévoir :

- l'authentification ;
- TLS ;
- la gestion sécurisée des secrets ;
- la restriction des accès réseau ;
- le contrôle des droits ;
- la protection des repositories de sauvegarde ;
- le chiffrement des sauvegardes si nécessaire ;
- une politique de rétention ;
- une séparation entre les données actives et les sauvegardes.

---

## 14. Points de contrôle avant une mise à jour

Avant une mise à jour importante, la procédure recommandée est :

1. vérifier l'état Git du projet ;
2. identifier le commit actuellement déployé ;
3. vérifier que la CI est verte ;
4. vérifier le Quality Gate SonarQube ;
5. consulter les Dependency Risks ;
6. sauvegarder les données persistantes concernées ;
7. vérifier l'archive de sauvegarde ;
8. appliquer la mise à jour dans une branche dédiée ;
9. exécuter les tests ;
10. construire les images Docker ;
11. tester avec Docker Compose ;
12. contrôler les logs dans Kibana ;
13. publier les nouvelles images uniquement après validation.

---

## 15. Points de contrôle après une mise à jour

Après une mise à jour ou un redéploiement :

1. vérifier l'état des conteneurs avec :

```powershell
docker compose ps
```

2. vérifier Elasticsearch avec :

```powershell
curl.exe "http://localhost:9200/_cluster/health?pretty"
```

3. vérifier Kibana avec :

```powershell
curl.exe -s http://localhost:5601/api/status
```

4. vérifier les logs Orion avec :

```powershell
curl.exe "http://localhost:9200/orion-logs-*/_count?q=service:orion-backend"
```

5. vérifier le backend sur le port `8080` ;

6. vérifier le frontend via les ports `8081` et `8443` ;

7. contrôler GitHub Actions et SonarQube Cloud.

En cas d'échec, le rollback doit être privilégié plutôt qu'une modification non contrôlée directement sur l'environnement.

---

## 16. Résultat du test de sauvegarde et restauration

Le test réalisé dans le cadre du projet a permis de valider la chaîne suivante :

```text
Volume Elasticsearch original
        |
        v
Arrêt d'Elasticsearch
        |
        v
Création de l'archive .tar.gz
        |
        v
Vérification du contenu de l'archive
        |
        v
Création d'un volume temporaire
        |
        v
Restauration de l'archive
        |
        v
Démarrage d'Elasticsearch sur le port 9201
        |
        v
Vérification de la santé du cluster
        |
        v
Recherche des logs Orion
        |
        v
59 documents retrouvés
        |
        v
Suppression de l'environnement temporaire
        |
        v
Redémarrage de la stack originale
        |
        v
59 documents toujours présents
        |
        v
Kibana disponible
```

La restauration a donc été vérifiée avec les données réelles de monitoring du projet et non uniquement par l'existence d'un fichier de sauvegarde.

---

## 17. Conclusion

Le projet Orion dispose désormais d'une procédure de sauvegarde et de restauration réellement testée pour les données persistantes Elasticsearch de la stack de monitoring.

Le test a démontré :

- l'arrêt contrôlé d'Elasticsearch ;
- la création d'une archive du volume persistant ;
- la vérification du contenu de l'archive ;
- la restauration dans un volume indépendant ;
- la présence de 29 répertoires d'index ;
- le démarrage d'une instance Elasticsearch restaurée ;
- la présence de 29 shards primaires actifs ;
- l'absence de shard primaire non assigné ;
- la récupération et l'interrogation de 59 logs du backend Orion ;
- la suppression sûre de l'environnement temporaire ;
- la conservation du volume original ;
- le redémarrage de la stack normale ;
- la conservation des 59 documents dans l'environnement original ;
- le retour de Kibana à l'état `available`.

Cette procédure est complétée par une stratégie de maintenance des dépendances, une approche de rollback basée sur les SHA Git et les images Docker publiées, ainsi qu'un plan périodique de contrôle.

Les valeurs RPO et RTO ne sont pas inventées : elles devront être définies et mesurées selon les exigences d'un futur environnement de production.