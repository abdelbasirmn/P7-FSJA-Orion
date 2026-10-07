# Indicateurs CI/CD et métriques DORA — Orion MicroCRM

## 1. Objectif

Ce document présente les indicateurs mesurés sur la chaîne CI/CD du projet Orion MicroCRM.

Les données utilisées proviennent des exécutions réelles du workflow GitHub Actions `ci.yml` sur la branche `main`.

L'objectif est de suivre la fiabilité et la durée de la chaîne CI/CD tout en distinguant les indicateurs réellement mesurables des métriques DORA qui nécessitent un environnement de production et un historique d'incidents.

## 2. Périmètre de mesure

L'échantillon étudié contient 10 exécutions du workflow CI réalisées les 6 et 7 octobre 2026.

La durée d'une exécution est calculée entre le démarrage réel du workflow (`run_started_at`) et sa dernière mise à jour après exécution (`updated_at`).

| Commit | Résultat | Durée |
|---|---|---:|
| `cf0ee15` | Succès | 126 s |
| `cd7a851` | Échec | 109 s |
| `09a01e9` | Succès | 131 s |
| `d9a81dd` | Succès | 180 s |
| `a640801` | Succès | 176 s |
| `a875766` | Succès | 228 s |
| `f2d0f16` | Succès | 215 s |
| `9a5393f` | Succès | 264 s |
| `c588559` | Succès | 389 s |
| `8f0e6ea` | Succès | 392 s |

## 3. KPI de la chaîne CI/CD

Sur les 10 exécutions observées :

- nombre total d'exécutions : 10 ;
- exécutions réussies : 9 ;
- exécutions échouées : 1 ;
- taux de réussite : 90 % ;
- durée moyenne : 221 secondes, soit 3 min 41 s ;
- durée minimale : 109 secondes, soit 1 min 49 s ;
- durée maximale : 392 secondes, soit 6 min 32 s.

Ces valeurs correspondent uniquement à l'échantillon observé et ne constituent pas des objectifs de niveau de service.

## 4. Interprétation

Le taux de réussite observé de 90 % indique que 9 des 10 exécutions étudiées se sont terminées avec succès.

L'unique échec de l'échantillon correspond au commit `cd7a851`. Les exécutions suivantes ont permis de poursuivre la mise en place et la validation de la chaîne CI/CD.

L'augmentation de la durée des exécutions les plus récentes s'explique notamment par l'évolution du pipeline. Celui-ci réalise désormais les constructions et tests, l'analyse SonarQube Cloud et, hors pull request, la publication des images Docker dans GitHub Container Registry.

## 5. Positionnement par rapport aux métriques DORA

Les quatre métriques DORA classiques sont :

1. Deployment Frequency ;
2. Lead Time for Changes ;
3. Change Failure Rate ;
4. Mean Time to Restore (MTTR).

### 5.1 Deployment Frequency

Le workflow publie les images Docker du backend et du frontend dans GitHub Container Registry après validation du job `build-test` pour les événements concernés hors pull request.

Cependant, la publication d'une image Docker dans un registre ne constitue pas à elle seule un déploiement en production.

La fréquence de déploiement en production n'est donc pas mesurable de manière fiable avec les données actuellement disponibles.

### 5.2 Lead Time for Changes

Les données collectées permettent de mesurer la durée d'exécution du pipeline, mais elles ne fournissent pas à elles seules le délai complet entre une modification du code et son déploiement effectif en production.

Le Lead Time for Changes DORA n'est donc pas calculé dans ce projet à ce stade.

### 5.3 Change Failure Rate

Un échec de workflow CI ne doit pas être assimilé automatiquement à un échec de déploiement en production.

Aucun historique de déploiements de production et d'incidents associés n'étant disponible, le Change Failure Rate de production n'est pas calculable de manière fiable.

Le taux de réussite CI de 90 % présenté dans ce document reste un KPI de pipeline distinct du Change Failure Rate DORA.

### 5.4 Mean Time to Restore (MTTR)

Aucun incident de production avec heure de début et heure de restauration n'a été enregistré dans le périmètre du projet.

Le MTTR ne peut donc pas être calculé sans inventer de données.

Dans un environnement de production, le MTTR serait calculé à partir des incidents réellement détectés et de leur temps de restauration.

## 6. Améliorations pour un suivi DORA complet

Pour mesurer les quatre métriques DORA dans un environnement réel, la chaîne pourrait être complétée par :

- l'enregistrement automatique de chaque déploiement de production ;
- l'association des commits aux déploiements ;
- la conservation de l'heure de début et de fin des déploiements ;
- l'enregistrement des incidents de production ;
- l'identification des déploiements ayant provoqué un incident ;
- l'enregistrement de l'heure de restauration du service ;
- la génération périodique d'un tableau de bord DORA.

## 7. Conclusion

L'échantillon étudié montre un taux de réussite CI de 90 % et une durée moyenne de pipeline de 221 secondes.

Ces indicateurs fournissent une première base de suivi de la performance de la chaîne CI/CD Orion.

Les métriques DORA liées aux déploiements et aux incidents de production ne sont volontairement pas inventées. Elles devront être calculées lorsque le projet disposera d'un environnement de production et d'un historique suffisant de déploiements et d'incidents.