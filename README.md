# Cabinet Médical — dépôt GitOps

Source de vérité de ce qui tourne sur le cluster. ArgoCD lit ce dépôt et synchronise l'état du cluster avec son contenu : **aucun `kubectl apply` ni `helm install` manuel**.

Projet complet : [cabinet-medical-app](https://github.com/rjafallah/cabinet-medical-app) (code et CI) · [cabinet-medical-infra](https://github.com/rjafallah/cabinet-medical-infra) (Terraform).

## Contenu

```
argocd/
├── app-dev.yaml                 # Application : l'appli Spring Boot + MySQL (dev)
├── kube-prometheus-stack.yaml   # Prometheus + Grafana
└── loki.yaml                    # Loki + Promtail
chart/                           # chart Helm cabinet-medical
├── Chart.yaml
├── values.yaml
├── files/base_donnees.sql       # script d'initialisation MySQL
└── templates/
    ├── app-deployment.yaml
    ├── app-service.yaml
    ├── mysql-configmap.yaml
    ├── mysql-service.yaml
    └── mysql-statefulset.yaml
envs/dev/values.yaml             # image.repository et image.tag (tag mis à jour par la CI)
```

## Fonctionnement

1. La CI du dépôt `app` construit l'image et commite le nouveau `image.tag` dans `envs/dev/values.yaml`.
2. ArgoCD détecte le commit, rend le chart avec ces valeurs et applique le résultat dans le namespace `dev`.
3. `selfHeal` annule toute modification manuelle ; `prune` supprime les ressources retirées de Git.

**Ordre de déploiement (sync waves)** : MySQL (vague 0, avec `readinessProbe`) avant l'application (vague 1), pour éviter les redémarrages de l'application au démarrage.

## Démarrage

Le bootstrap du cluster (dépôt infra) installe ArgoCD puis applique les trois fichiers de `argocd/`. Rien d'autre à faire.

Les `Application` ne sont **pas** surveillées par ArgoCD (pas d'App of Apps) : après modification d'un fichier de `argocd/`, le réappliquer sur un cluster existant :

```bash
B=https://raw.githubusercontent.com/rjafallah/cabinet-medical-gitops/main/argocd
kubectl apply -f $B/app-dev.yaml -f $B/kube-prometheus-stack.yaml -f $B/loki.yaml
```

## Accès aux interfaces

Aucun port n'est ouvert sur Internet. ArgoCD et Grafana sont exposés en NodePort sur le nœud et joints par tunnel SSH :

```bash
ssh -i ~/.ssh/<clé> -L 8443:localhost:30443 -L 3000:localhost:30300 ubuntu@<IP>
```

| Interface | URL | Identifiants |
|---|---|---|
| ArgoCD | <https://localhost:8443> | `admin` / `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' \| base64 -d` |
| Grafana | <http://localhost:3000> | `admin` / `admin` |

## Observabilité

- **Métriques** : un `ServiceMonitor` (défini dans les valeurs de `kube-prometheus-stack`) scrape `/actuator/prometheus`. Dashboard *JVM (Micrometer)* importé automatiquement (n° 4701 de grafana.com).
- **Logs** : Promtail collecte les logs des pods, Loki les stocke, la datasource Loki est ajoutée à Grafana. Requête type : `{namespace="dev"}`.

## Rollback

```bash
git log --oneline -5          # repérer le commit "ci: bump image tag to <sha>"
git revert <hash>
git push                      # ArgoCD resynchronise sur l'image précédente
```

## Limites

- Identifiants MySQL de test en clair dans le chart (Sealed Secrets non mis en place).
- Un seul environnement (`dev`) et un seul nœud.
- Pas de règles d'alerte, Alertmanager désactivé.
- `loki-stack` est un chart déprécié.
