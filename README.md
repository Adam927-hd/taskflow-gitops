# taskflow-gitops — dépôt GitOps du cours CI/CD M2

Ce dépôt décrit **l'état voulu** de l'application TaskFlow dans Kubernetes.
Argo CD le surveille et aligne le cluster dessus : pour changer la production,
on ne tape pas de commande, on fait une **Pull Request**.

## Sommaire

- [Installation](#installation-à-faire-chez-vous-avant-le-cours) · [Structure](#structure) · [Images disponibles](#images-disponibles) · [Équipe](#équipe)
- **Lab 1 : déployer par PR, dérive, retour arrière**
  - [Livrable 1](#livrable-lab-1) : [Push ou Pull ?](#push-ou-pull-) · [Qui a corrigé quoi ?](#qui-a-corrigé-quoi-) · [Pourquoi `git revert` ?](#pourquoi-git-revert-)
  - [Étapes et captures](#lab-1)
- **Lab 2 : Blue-Green puis Canary**
  - [Livrable 2](#livrable-2) : Blue-Green ou Canary pour TaskFlow ?
  - [A • Blue-Green](#a--blue-green) · [B • Canary](#b--canary)
- **Lab 3 : analyse automatique, abort, postmortem**
  - [Livrable 3](#livrable-3) : AnalysisRun en échec (2.1.0) et en succès (2.2.0)
  - [A • L'étalon et l'analyse](#a--létalon-et-lanalyse) · [B • L'incident](#b--lincident)
  - [Postmortem 2.1.0](postmortem-2.1.0.md)

## Installation (à faire chez vous, avant le cours)

Prérequis : Docker Desktop démarré, 8 Go de RAM, 10 Go de disque libre.
Sous Windows : WSL2 (Ubuntu) + intégration WSL de Docker Desktop, et toutes les commandes dans WSL.

```bash
git clone https://github.com/9m7fjfpv9k-cyber/taskflow-gitops.git
cd taskflow-gitops
./scripts/install.sh
```

Le script crée un cluster local `kind`, installe Argo CD et Argo Rollouts,
puis télécharge les images des labs. Comptez 5 à 15 minutes.
Il peut être relancé sans risque.

## Structure

| Chemin                    | Rôle                                                            |
| ------------------------- | --------------------------------------------------------------- |
| `apps/taskflow/`          | Les manifests surveillés par Argo CD                            |
| `argocd/application.yaml` | Déclare l'application dans Argo CD                              |
| `exemples/bluegreen/`     | Manifests pour le déploiement Blue-Green                        |
| `exemples/canary/`        | Manifests pour le déploiement Canary                            |
| `exemples/robustesse/`    | Canary avec test de charge k6 automatique, modèle de postmortem |
| `exemples/ci/`            | Pipelines de la mini-PSSI (GitHub Actions et GitLab CI)         |
| `policies/`               | Mini-PSSI et règles Rego vérifiées par conftest                 |
| `scripts/install.sh`      | Installation de l'environnement                                 |
| `scripts/argocd-ui.sh`    | Ouvre l'interface d'Argo CD                                     |
| `scripts/observe.sh`      | Montre quelle version répond, et avec quel code HTTP            |
| `scripts/charge.sh`       | Lance à la main le test de charge k6 contre un service          |

## Images disponibles

`ghcr.io/9m7fjfpv9k-cyber/taskflow` en versions `1.0.0`, `1.1.0`, `2.0.0`, `2.1.0` et `2.2.0`.

## Équipe

<!-- Noms du binôme -->

- Nicolas, Adam

## Livrable Lab 1

Les déploiements sont déclenchés par les modifications du dépôt Git.
Argo CD surveille le dépôt et synchronise l'état du cluster avec la configuration déclarée.
Chaque déploiement à une Pull Request (PR) sur main, ce qui permet de retrouver l'historique des changements.

### Push ou Pull ?

Nous utilisons une approche **Pull** : Argo CD récupère les modifications depuis le dépôt Git et les applique au cluster Kubernetes.
Contrairement à une approche Push, le pipeline CI ne déploie pas directement les ressources sur le cluster. Argo CD surveille le dépôt et assure la synchronisation.

### Qui a corrigé quoi ?

Les modifications sont traçables grâce à Git :

- Les commits identifient les auteurs et les changements.
- Les Pull Requests permettent de consulter les modifications proposées et les discussions.
- L'historique Git permet de retrouver les corrections et leurs motivations.

Cette traçabilité facilite la collaboration et l'identification des responsabilités.

### Pourquoi `git revert` ?

`git revert` permet d'annuler un changement en créant un nouveau commit qui inverse les modifications précédentes.

Dans notre cas, le bouton **Revert** de la Pull Request permet de revenir à la version `1.0.0` de l'image en annulant le changement vers `2.0.0`.

Cette méthode conserve l'historique Git et permet à Argo CD de détecter le changement, puis de redéployer la version précédente.

## LAB 1

1. Forker taskflow-gitops, inviter le binôme, ruleset sur main

2. argocd/application.yaml : votre pseudo, puis kubectl apply (chacun sur son cluster)

![Argo UI](captures/argo.png)

Argo en localhost
→ Argo CD est installé et accessible : c'est lui qui déploiera à notre place.

![Changement du pseudo](captures/pseudo.png)

Mise a jour du repoURL dans application.yaml
→ Argo CD doit surveiller notre fork, pas le dépôt du formateur.

![Configuration du contexte](captures/kubectl.png)

Changement du contexte et apply de taskflow
→ On cible `kind-cicd` pour ne pas déployer sur un autre cluster (AKS, k3d).

3. Vérifier Synced + Healthy, puis ./scripts/observe.sh

![get app terminal](captures/synced.png)

Vérification Synced + Healthy sur le terminal
→ Synced : le cluster correspond à Git. Healthy : les pods fonctionnent.

![Argo UI healthy](captures/healthy.png)

Vérification Synced + Healthy sur le dashboard

![Observe](captures/observe.png)

On observe bien la version 1.0.0 avec le script
→ 40 réponses http=200 : l'application répond réellement, pas seulement « Running ».

4. PR : image 2.0.0. Mesurer le délai jusqu'au déploiement

![Version 2.0.0 sur argo ui](captures/image2.png)

Version 2.0.0 sur argo
→ Le merge de la PR suffit : aucune commande de déploiement n'a été lancée.

![Watch des pods pendant la merge](captures/delaimerge.png)

Watch des pods pendant la merge
→ Les pods sont remplacés progressivement (rolling update), sans coupure.

![Watch pour voir combien de temps s'est écoulé apres la fin de la merge](captures/mesuredelai.png)

Watch pour voir combien de temps s'est écoulé apres la fin de la merge
→ Le délai vient du mode pull : Argo CD relit Git toutes les 60 s.

5. Dérive : kubectl scale, kubectl set image. Que fait Argo CD ?

**Scale**

![Scale d'un replica](captures/scale2.png)

Commende pour scale 1 replicas
→ On crée volontairement une dérive : le cluster ne correspond plus à Git (4 réplicas).

![Watch pendant le scale](captures/scale1.png)

On observe que les 3 réplicas delete on été relancé automatiquement
→ `selfHeal: true` : Argo CD détecte l'écart et réapplique Git.

**Set image**

![Lancement de la commande pour v1.0.0](captures/setimage2.png)

Commande pour passer a la version 1.0.0
→ Deuxième dérive : l'image du cluster diffère de celle déclarée dans Git.

![Watch pendant le set image](captures/setimage1.png)

On observe une erreur
→ Argo CD annule la modification pendant qu'elle s'applique.

![Observe pour verifier la version](captures/setimage2.png)

La version n'a pas changé
→ C'est toujours la version de Git qui tourne.

On peut conclure que Git reste la source de vérité. Les modifications manuelles du cluster ne remplacent pas la configuration déclarée dans le dépôt.

6. Bouton Revert sur la PR : retour en 1.0.0

![Observe avant/apres le revert](captures/revert.png)

Observe avant/apres le revert
→ Le retour en 1.0.0 est confirmé par les réponses réelles du service.

![Watch pendant le revert](captures/revert2.png)

Watch pendant le revert, les pods ont redémarré
→ Le revert est un commit comme un autre : Argo CD le déploie automatiquement.

![Git log](captures/revert4.png)

On observe bien un revert de la version 2.0.0
→ L'historique est conservé : on sait qui a déployé et qui est revenu en arrière.

Cette méthode conserve l'historique Git et permet à Argo CD de détecter le changement, puis de redéployer la version précédente.

7. Bonus : supprimer service.yaml par PR (prune)

![Suppresion de service avec git rm](captures/suppression.png)
Suppresion de `service.yaml` avec git rm
→ La suppression passe par une PR, comme tout autre changement.

![Observation](captures/prune.png)
On observe que supprimer un fichier de configuration dans Git peut entraîner la suppression de la ressource correspondante dans Kubernetes
→ C'est l'effet de `prune: true`. Sans lui, le Service resterait dans le cluster, orphelin.

## Livrable 2

Canary pour TaskFlow.

Risque : avec la 2.1.0 défectueuse, Canary n'a exposé que 15 % du trafic (6 erreurs 500 sur 40), contre environ 60 % d'erreurs pour tous les utilisateurs avec Blue-Green. L'abort a rétabli la 2.0.0 instantanément.
Coût : Blue-Green double les pods (8 au lieu de 4) pendant la transition. Canary n'en ajoute presque pas, mais le déploiement est plus lent.

## LAB 2

### A • Blue-Green

1. Remplacer deployment.yaml par exemples/bluegreen/ (PR)

![Git diff](<captures/1_Capture d’écran 2026-10-14 à 14.50.03.png>)

Ajout de la stratégie Bluegreen avec déploiement
→ Le Rollout remplace le Deployment : même template, seule la stratégie change.

2. PR: image 1.1.0

`image: ghcr.io/9m7fjfpv9k-cyber/taskflow:1.1.0`

3. observe.sh sur taskflow et taskflow-preview, compter les pods

![observe taskflow / observe taskdlow preview](<captures/2_Capture d’écran 2026-10-14 à 15.15.16.png>)

Observe taskflow / Observe taskdlow-preview
On observe que la preview passe à la version 1.1.0
→ La production reste en 1.0.0 : on peut tester la 1.1.0 sans exposer les utilisateurs.

![get pods](<captures/3_Capture d’écran 2026-10-14 à 15.15.39.png>)

Les pods sont doublés
→ 8 pods au lieu de 4 : c'est le coût de Blue-Green pendant la transition.

4. promote, puis observe.sh

![Commande promote](<captures/4_Capture d’écran 2026-10-14 à 15.17.55.png>)

La bascule reste une décision humaine (`autoPromotionEnabled: false`).

![Promote observe](<captures/5_Capture d’écran 2026-10-14 à 15.18.14.png>)

Lorsque la commande promote a été lancé on observe que que taskflow passe aussi en 1.1.0
→ La bascule est totale et instantanée : 100 % du trafic change de version d'un coup.

### B • Canary

1. exemples/canary/rollout.yaml, image 1.1.0 (PR)

![rollout de cnaary](<captures/Capture d’écran 2026-10-14 à 16.29.50.png>)

![v1.1.0](<captures/Capture d’écran 2026-10-14 à 16.31.02.png>)

On change de stratégie en gardant l'image 1.1.0 déjà en production : aucune nouvelle version n'est déployée.

2. PR : image 2.0.0, suivre avec --watch et observe.sh

![observe et watch](<captures/6_Capture d’écran 2026-10-14 à 15.49.45.png>)

Dans le observe on voit que la version 2.0.0 a démarré
Dans le watch on observe que 25% (1 sur 4) du traffic est mise a jour
→ Sans maillage de service, la part de trafic suit la proportion de pods. Le rollout attend une promotion manuelle.

3. promote jusqu'à 100 %

![promote 50%](<captures/7_Capture d’écran 2026-10-14 à 15.50.46.png>)

50% (2 pods sur 4)

![promote 75%](<captures/8_Capture d’écran 2026-10-14 à 15.52.31.png>)

75% (3 pods sur 4)

![promote  100%](<captures/9_Capture d’écran 2026-10-14 à 15.53.33.png>)

100% (4 pods sur 4)
→ Après la première promotion, les paliers suivants s'enchaînent seuls (pauses de 60 s puis 30 s).

4. PR : image 2.1.O. Regardez les codes HTTP. Puis abort... et après ?

![image 2.1.0](<captures/10_Capture d’écran 2026-10-14 à 16.00.23.png>)

A l'ajoute de l'image 2.1.0 -> Status en pause + http=500
→ La 2.1.0 est défectueuse, mais seuls 25 % du trafic y sont exposés.

![abort cmd](<captures/Capture d’écran 2026-10-14 à 16.35.11.png>)

L'abort renvoie immédiatement tout le trafic vers la version stable 2.0.0.

![degraded](<captures/11_Capture d’écran 2026-10-14 à 16.04.25.png>)

Apres l'abort le status passe en dégradé
→ Et après ? Git demande toujours la 2.1.0 : il faut faire un revert de la PR pour que Git redevienne la source de vérité.

## Livrable 3

![AnalysisRun 2.1.0 en échec, 2.2.0 en succès](captures/postmortem/B4_5.png)

- `taskflow-df976ccb5-6-2` (2.1.0) : **Failed**. k6 mesure 32 % d'erreurs et un p95 de 313 ms, au-delà des seuils (< 2 % et < 250 ms). Argo Rollouts abandonne seul le déploiement.
- `taskflow-7ddd57d788-8-2` (2.2.0) : **Successful**. Les seuils sont respectés et le Rollout continue jusqu'à 100 %.

→ L'analyse remplace la décision humaine : seule une version qui passe le test de charge dépasse 25 %. Détails dans [le postmortem](postmortem-2.1.0.md).

## Lab 3

### A • L'étalon et l'analyse

1. Sync fork, git pull, production en 2.0.0

![PR retour en 2.0.0](<captures/postmortem/A1 PR.png>)

Retour de l'image 2.1.0 à 2.0.0 par PR
→ La production doit partir d'une version saine pour servir de référence.

2. ./scripts/charge.sh http://taskflow → noter erreurs et p95

![Test de charge en cours](captures/postmortem/A2_0.png)

Test k6 : 5 utilisateurs virtuels pendant 30 s sur le service de production

![Résultat de l'étalon](captures/postmortem/A2_1.png)

Étalon de la 2.0.0 : **0,00 % d'erreurs, p95 = 10,05 ms**
→ Ce sont nos valeurs de référence. Les seuils du test sont < 2 % d'erreurs et un p95 < 250 ms.

3. PR feat/analyse-auto : supprimer deployment.yaml, copier les 4 fichiers de exemples/robustesse/

![Fichiers de robustesse](captures/postmortem/A3.png)

`analysis-template.yaml`, `configmap-k6.yaml`, `rollout.yaml` et `service-canary.yaml` sont copiés dans `apps/taskflow`
→ Le Rollout lance désormais k6 automatiquement au palier de 25 %.

4. apps/taskflow : un Rollout, aucun Deployment. Vérifier les CRD et get analysistemplate,configmap,svc

![Vérifications](captures/postmortem/A4.png)

Rollout `Healthy` en 2.0.0, CRD Argo présentes, aucun Deployment, AnalysisTemplate `robustesse-k6`, ConfigMap `k6-robustesse` et Service `taskflow-canary` créés
→ Sans les CRD, Argo CD refuserait les ressources `AnalysisTemplate`. Le service `taskflow-canary` permet de tester uniquement la nouvelle version.

### B • L'incident

1. PR : image 2.1.0. --watch, et ne touchez à rien

![Démarrage du canary 2.1.0](captures/postmortem/B1_0.png)

Le pod 2.1.0 démarre (révision 6), objectif 25 %

![AnalysisRun en cours](captures/postmortem/B1_1.png)

Au palier de 25 %, après 20 s de pause, l'AnalysisRun lance le Job k6 sur `taskflow-canary`

![AnalysisRun en échec](captures/postmortem/B1_2.png)

L'AnalysisRun passe en **Failed**, puis le Rollout passe en **Degraded** : `RolloutAborted ... failed (1) > failureLimit (0)`
→ Abort automatique : aucune intervention humaine.

![Retour en 2.0.0](captures/postmortem/B1_3.png)

![4 pods 2.0.0](captures/postmortem/B1_4.png)

Le pod 2.1.0 est supprimé et les 4 pods repassent en 2.0.0
→ Le trafic revient sur la version stable en moins d'une minute.

2. Enquête : réunir les 4 preuves

**Preuve 1 : l'AnalysisRun**

![describe analysisrun](captures/postmortem/B2_1.png)

![describe analysisrun, statut](captures/postmortem/B2_2.png)

`kubectl describe analysisrun` : cible `http://taskflow-canary`, phase `Failed`, de 09:23:48 à 09:24:25 UTC

**Preuve 2 : les logs k6**

![logs k6](captures/postmortem/B2_3.png)

![logs k6, suite](captures/postmortem/B2_4.png)

![logs k6, seuils](captures/postmortem/B2_5.png)

Les deux seuils sont dépassés : **32,33 % d'erreurs** (97/300) et **p95 = 312,76 ms**
→ Comparée à l'étalon (0 % et 10 ms), la 2.1.0 est environ 30 fois plus lente sur `/tasks` et renvoie 1 erreur sur 3.

**Preuve 3 : les événements**

![events](captures/postmortem/B2_6.png)

Chronologie complète : passage à 25 %, pause de 20 s, lancement de l'analyse, `MetricFailed`, `RolloutAborted`, retour à 4 pods 2.0.0
→ Les readiness probes ne testent que `/health`, qui répondait correctement. Seul le test de charge a détecté le problème.

3. Revert de la PR 2.1.0

![PR revert](captures/postmortem/B3_1.png)

PR #16 : revert de la 2.1.0, mergée

![Rollout healthy](captures/postmortem/B3_2.png)

![Révision 7](captures/postmortem/B3_3.png)

![get rollout](captures/postmortem/B3_4.png)

Le Rollout repasse **Healthy** (révision 7) en 2.0.0
→ L'abort avait protégé le cluster, mais Git demandait toujours la 2.1.0. Le revert remet Git et le cluster en accord.

![describe après revert](captures/postmortem/B3_5.png)

![events après revert](captures/postmortem/B3_6.png)

L'AnalysisRun en échec reste visible dans l'historique : `Rollback to stable ReplicaSets`, puis `Rollout updated to revision 7`

4. PR : image 2.2.0, suivre jusqu'à 100 %

![PR 2.2.0](captures/postmortem/B4_1.png)

PR #17 : image 2.0.0 → 2.2.0

![Canary 2.2.0](captures/postmortem/B4_2.png)

![Analyse 2.2.0 en cours](captures/postmortem/B4_3.png)

Même scénario : 25 %, puis AnalysisRun `taskflow-7ddd57d788-8-2`

![100 % en 2.2.0](captures/postmortem/B4_4.png)

AnalysisRun **Successful**, puis 50 %, 75 % et 100 % automatiquement, avec 4 pods 2.2.0
→ La version passe le même contrôle que la 2.1.0 et peut aller jusqu'à 100 %.

![observe et analysisrun](captures/postmortem/B4_5.png)

`observe.sh` montre la bascule en cours (34 × 2.0.0, 6 × 2.2.0, tout en http=200). On voit côte à côte l'AnalysisRun de la 2.1.0 (Failed) et celle de la 2.2.0 (Successful).

![describe analysisrun 2.2.0](captures/postmortem/B4_6.png)

![describe analysisrun 2.2.0, statut](captures/postmortem/B4_7.png)

![logs k6 2.2.0](captures/postmortem/B4_8.png)

![logs k6 2.2.0, suite](captures/postmortem/B4_9.png)

![logs k6 2.2.0, seuils](captures/postmortem/B4_10.png)

Résultat k6 sur la 2.2.0 : **0,00 % d'erreurs, p95 = 12,67 ms**
→ Des valeurs proches de l'étalon de la 2.0.0 : pas de régression.

![events 2.2.0](captures/postmortem/B4_11.png)

Événements : `AnalysisRunSuccessful`, puis paliers 50 % et 75 % jusqu'à la révision 8 complète

5. docs/postmortem-2.1.0.md

Voir [le postmortem de l'incident 2.1.0](postmortem-2.1.0.md).
