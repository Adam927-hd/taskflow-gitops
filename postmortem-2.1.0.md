# Postmortem — TaskFlow 2.1.0 lente et en erreur sur /tasks

> Sans reproche : on cherche ce qui a permis l'erreur, pas qui l'a faite.

| Champ | Valeur |
| --- | --- |
| Date et heure | 15/10/2026, 09:23 → 09:24 UTC (11:23 → 11:24 heure de Paris) |
| Version en cause | `ghcr.io/9m7fjfpv9k-cyber/taskflow:2.1.0` |
| PR à l'origine | #15 `feat/image-2.1.0` (image 2.0.0 → 2.1.0 dans `apps/taskflow/rollout.yaml`) |
| Durée d'exposition | ≈ 1 min : du passage à 25 % (≈ 09:23:28) à l'abort (09:24:25) |
| Part du trafic touché | 25 % (1 pod sur 4), dont environ 1 requête sur 3 en erreur → ≈ 8 % des requêtes en échec |
| Détecté par | Test de charge k6 automatique (AnalysisRun `taskflow-df976ccb5-6-2`) |
| Résolu par | Abort automatique d'Argo Rollouts, puis `git revert` (PR #16) |

## Chronologie

| Heure (UTC) | Événement |
| --- | --- |
| avant | Étalon sur la 2.0.0 (`./scripts/charge.sh http://taskflow`) : **0,00 % d'erreurs, p95 = 10,05 ms** |
| ≈ 09:23:08 | Merge de la PR #15, Argo CD applique la 2.1.0 → Rollout révision 6 |
| ≈ 09:23:28 | Palier `setWeight: 25` : 1 pod 2.1.0, le service `taskflow-canary` bascule dessus |
| 09:23:48 | Fin de la pause de 20 s, lancement de l'AnalysisRun et du Job k6 sur `http://taskflow-canary` |
| 09:24:19 | k6 termine : `thresholds on metrics 'http_req_duration, http_req_failed' have been crossed` |
| 09:24:25 | AnalysisRun **Failed** (`failed (1) > failureLimit (0)`), puis **RolloutAborted** : retour à 4 pods 2.0.0, Rollout `Degraded` |
| ≈ 09:41 | Merge de la PR #16 (revert de la 2.1.0) → Rollout révision 7, `Healthy`, 100 % 2.0.0 |
| ensuite | PR #17 : image 2.2.0, AnalysisRun `taskflow-7ddd57d788-8-2` **Successful**, déploiement jusqu'à 100 % |

![Abort automatique](../captures/postmortem/B1_3.png)

## Composant défaillant et cause racine

- **Quel composant a échoué ?** L'endpoint `/tasks` de l'image 2.1.0. Preuve : `kubectl -n taskflow logs job/0a71779f-…test-de-charge-k6.1`.

  | Mesure k6 (30 s, 5 VUs) | 2.0.0 (étalon) | 2.1.0 (canary) | Seuil |
  | --- | --- | --- | --- |
  | `http_req_failed` | 0,00 % | **32,33 %** (97/300) | < 2 % |
  | `http_req_duration` p95 | 10,05 ms | **312,76 ms** | < 250 ms |
  | Débit | 24,2 req/s | 9,8 req/s | — |

  ![Logs k6 de l'AnalysisRun](../captures/postmortem/B2_5.png)

- **Pourquoi les probes Kubernetes ne l'ont-elles pas vu ?** Les probes `readiness` et `liveness` n'interrogent que `/health`, qui répondait correctement. Le pod 2.1.0 était donc `Running 1/1`. Les probes vérifient que le processus est vivant, pas que le service rend bien son travail ni dans quels délais.
- **Cause racine :** une régression de la 2.1.0 sur `/tasks` (environ +300 ms par requête et une requête sur trois en erreur) est arrivée en production sans test de performance avant le merge. Seule l'analyse k6 au palier de 25 % pouvait la détecter.

## Ce qui a bien fonctionné

- **Le Canary** : la 2.1.0 n'a reçu que 25 % du trafic.
- **L'analyse automatique** : k6 a mesuré le canary seul, grâce au service `taskflow-canary`, et a comparé le résultat à nos seuils. L'abort a eu lieu en environ 1 min, **sans intervention humaine**.
- **GitOps** : le revert de la PR a remis Git en cohérence avec le cluster, et l'historique garde la trace de l'incident.
- **La version 2.2.0** a passé la même analyse (AnalysisRun Successful) avant d'atteindre 100 %.

## Actions correctives

| Action | Responsable | Échéance |
| --- | --- | --- |
| Lancer `./scripts/charge.sh` sur la nouvelle image dans la CI, avant le merge | Binôme | J4 |
| Ajouter une readiness probe sur `/tasks` (ou un `/ready` qui teste les dépendances) | Dev TaskFlow | J4 |
| Garder l'analyse k6 à chaque palier du Canary (50 %, 75 %) et pas seulement à 25 % | Binôme | J4 |
| Prévenir l'équipe automatiquement en cas de `RolloutAborted` (notifications Argo) | Binôme | J5 |
