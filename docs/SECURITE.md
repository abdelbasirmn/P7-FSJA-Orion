# Analyse de sécurité et plan de remédiation — Orion MicroCRM

## 1. Objectif

Ce document présente l'état de sécurité observé sur le projet Orion MicroCRM à partir de l'analyse SonarQube Cloud et définit un plan de remédiation priorisé.

Les valeurs présentées correspondent aux résultats réellement observés lors de l'analyse du 7 octobre 2026. Elles constituent un instantané de l'état du projet et peuvent évoluer lors des analyses suivantes.

L'objectif est de distinguer :

- les issues détectées dans le code ;
- les Security Hotspots ;
- les risques liés aux dépendances ;
- les actions de remédiation à appliquer selon leur priorité.

## 2. État de l'analyse SonarQube Cloud

Lors de la vérification réalisée le 7 octobre 2026, SonarQube Cloud présente :

- 73 issues au total sur le code analysé ;
- 38 issues classées dans la vue Top issues ;
- aucun Security Hotspot restant à examiner ;
- 347 Dependency Risks ;
- 978 dépendances analysées.

Le Quality Gate du projet était également validé lors des analyses précédentes de la chaîne CI/CD.

Les issues de code, les Security Hotspots et les Dependency Risks représentent des catégories différentes et ne doivent pas être additionnés pour produire un nombre global de vulnérabilités.

## 3. Issues de code

La page Issues de SonarQube Cloud contient 73 issues.

La vue Top issues en présente 38 avec les filtres de priorité appliqués lors de la vérification.

Un exemple observé concerne le fichier :

`front/src/app/person-details/person-details.component.ts`

SonarQube recommande :

`Refactor this asynchronous operation outside of the constructor.`

Cette issue présente notamment des impacts élevés sur la fiabilité et la maintenabilité.

Ces issues doivent être analysées individuellement avant correction afin d'éviter d'assimiler automatiquement une issue de maintenabilité ou de fiabilité à une vulnérabilité de sécurité.

## 4. Security Hotspots

La page Security Hotspots indique :

- 100 % des Security Hotspots revus ;
- aucun Security Hotspot restant à examiner.

SonarQube Cloud indique également que le concept de Security Hotspots est déprécié dans cette interface et que les résultats concernés apparaissent désormais sous forme d'issues ou de vulnérabilités dans la page Issues.

L'absence de Security Hotspot à examiner ne signifie pas que l'ensemble du projet est exempt de risques de sécurité. Les risques liés aux dépendances doivent notamment être traités séparément.

## 5. Dependency Risks

SonarQube Cloud détecte 347 risques dans 978 dépendances.

La répartition observée par sévérité est la suivante :

| Sévérité | Nombre de risques |
|---|---:|
| Blocker | 2 |
| High | 7 |
| Medium | 149 |
| Low | 188 |
| Info | 1 |
| **Total** | **347** |

La somme des catégories correspond bien aux 347 risques affichés par SonarQube Cloud.

## 6. Exemples de risques observés

### 6.1 Risque Blocker sur Tomcat

Un risque Blocker observé concerne :

- CVE : `CVE-2025-24813` ;
- composant : `org.apache.tomcat.embed:tomcat-embed-core` ;
- version observée : `10.1.20` ;
- type : vulnérabilité ;
- CVSS affiché : 9.8, Critical ;
- dépendance : transitive.

Ce risque doit être traité en priorité.

Comme la dépendance est transitive, la remédiation doit commencer par l'identification de la dépendance directe qui introduit cette version de Tomcat avant de modifier la configuration Gradle.

### 6.2 Risque Blocker associé à Vite

Un second risque Blocker observé concerne :

- CVE : `CVE-2025-31125` ;
- composant observé : `vite` ;
- version observée : `5.1.7` ;
- type : vulnérabilité ;
- dépendance indiquée comme transitive dans l'analyse.

Ce risque doit également être analysé avant toute mise à jour du frontend.

### 6.3 Risques High

La catégorie High contient 7 risques.

Parmi les éléments visibles lors de l'analyse, certains concernent également `org.apache.tomcat.embed:tomcat-embed-core 10.1.20`.

Les risques High doivent être traités après les Blocker et avant les catégories Medium et Low.

## 7. Priorisation du plan de remédiation

### P1 — Risques Blocker

Priorité immédiate :

1. analyser les 2 risques Blocker ;
2. identifier les dépendances directes responsables des dépendances transitives ;
3. rechercher une version corrigée compatible ;
4. effectuer les mises à jour de manière contrôlée ;
5. reconstruire les applications ;
6. exécuter tous les tests backend et frontend ;
7. relancer SonarQube Cloud ;
8. vérifier la disparition ou la réduction des risques concernés.

Une mise à jour forcée globale des dépendances ne doit pas être utilisée sans validation de compatibilité.

### P2 — Risques High

Après traitement des Blocker :

1. analyser les 7 risques High ;
2. identifier les composants et versions concernés ;
3. évaluer leur exploitabilité dans le contexte Orion ;
4. appliquer les mises à jour ou mesures correctives compatibles ;
5. relancer les tests et l'analyse SonarQube.

### P3 — Risques Medium

Les 149 risques Medium doivent être intégrés au backlog technique.

Ils peuvent être regroupés par dépendance afin d'éviter plusieurs mises à jour indépendantes lorsqu'une seule montée de version permet de corriger plusieurs risques.

Chaque modification doit être accompagnée des tests appropriés.

### P4 — Risques Low et Info

Les 188 risques Low et le risque Info doivent faire l'objet d'un suivi périodique.

Ils sont moins prioritaires que les Blocker, High et Medium, mais doivent rester visibles dans le processus de maintenance des dépendances.

## 8. Stratégie de mise à jour des dépendances

La remédiation doit privilégier une approche progressive :

1. identifier la dépendance vulnérable ;
2. déterminer si elle est directe ou transitive ;
3. identifier le paquet parent lorsqu'elle est transitive ;
4. vérifier les versions corrigées disponibles ;
5. évaluer les éventuels changements incompatibles ;
6. mettre à jour une famille de dépendances de manière contrôlée ;
7. exécuter les tests ;
8. reconstruire les images Docker ;
9. relancer SonarQube Cloud ;
10. comparer les résultats avant et après correction.

Pour le frontend, une commande telle que `npm audit fix --force` ne doit pas être appliquée automatiquement, car elle peut introduire des changements majeurs et casser la compatibilité de l'application.

## 9. Intégration avec la chaîne CI/CD

L'analyse SonarQube Cloud est déjà intégrée au workflow GitHub Actions.

Le processus de sécurité s'inscrit donc dans la chaîne suivante :

`Commit -> Build -> Tests -> Couverture -> Analyse SonarQube -> Publication des images Docker`

Les corrections de sécurité doivent passer par la même chaîne de validation afin de vérifier qu'elles ne provoquent pas de régression.

À terme, le processus peut être renforcé par :

- une politique explicite de traitement des risques Blocker et High ;
- une revue périodique des Dependency Risks ;
- la mise à jour régulière des dépendances ;
- la conservation des résultats avant et après remédiation ;
- l'automatisation de contrôles supplémentaires de dépendances lorsque cela est pertinent.

## 10. Limites de l'analyse

Les résultats SonarQube constituent une aide à l'analyse de sécurité, mais ne remplacent pas une évaluation complète de l'application en production.

Un risque détecté dans une dépendance n'implique pas automatiquement que la vulnérabilité est exploitable dans le contexte de l'application.

Inversement, l'absence de Security Hotspot à examiner ne garantit pas l'absence de toute vulnérabilité.

Les résultats doivent donc être interprétés selon l'architecture, les fonctionnalités réellement utilisées et l'exposition de l'application.

## 11. Conclusion

L'analyse observée met principalement en évidence un enjeu de gestion des dépendances : 347 Dependency Risks sont recensés sur 978 dépendances.

La priorité de remédiation est constituée des 2 risques Blocker, suivis des 7 risques High. Les 149 risques Medium doivent ensuite être planifiés dans le backlog technique, tandis que les 188 Low et le risque Info doivent faire l'objet d'un suivi périodique.

Le projet dispose déjà d'une analyse SonarQube Cloud intégrée à la CI/CD. Les prochaines corrections devront être appliquées progressivement et validées par les builds, les tests et une nouvelle analyse SonarQube avant d'être considérées comme résolues.