\# TP4 — GitOps, Argo CD et observabilité



Ce TP déploie Podinfo sur un cluster Kubernetes local. Argo CD applique les

manifestes déclarés dans GitHub. Prometheus collecte les métriques de

l'application et d'Argo CD, puis Grafana les affiche dans un dashboard.



\## Architecture



```mermaid

flowchart LR

&#x20;   GitHub -->|Manifestes Kustomize| ArgoCD\["Argo CD"]

&#x20;   ArgoCD -->|Déploie et synchronise| Podinfo\["Podinfo sur Minikube"]

&#x20;   Prometheus -->|Collecte les métriques| Podinfo

&#x20;   Prometheus -->|Collecte les métriques| ArgoCD

&#x20;   Grafana -->|Requêtes PromQL| Prometheus

```



Argo CD gère trois applications :



\- `podinfo` déploie l'application dans le namespace `demo`, avec Auto-Sync

&#x20; et Self-Healing.

\- `monitoring` déploie `kube-prometheus-stack` en version `77.0.0` dans le

&#x20; namespace `monitoring`.

\- `monitoring-config` applique les quatre `ServiceMonitor` et la règle

&#x20; d'alerte depuis Git.



\## Structure du dépôt



| Chemin | Contenu |

|---|---|

| `apps/podinfo/base/` | Deployment et Service Podinfo |

| `apps/podinfo/overlays/tp/` | Nombre de répliques et tag d'image |

| `bootstrap/` | Définitions des trois Applications Argo CD |

| `monitoring/` | ServiceMonitor et règle d'alerte Podinfo |

| `dashboard/tp4-gitops.json` | Dashboard Grafana exporté |



\## Installation sous Windows



Prérequis : Git, Docker Desktop en mode Linux et Minikube. Laisser Docker

Desktop ouvert. Cloner le dépôt et lancer Minikube depuis PowerShell.



```powershell

git clone https://github.com/eyaabenslama/tp4-gitops.git

Set-Location tp4-gitops



minikube start --driver=docker --cpus=4 --memory=6144

minikube kubectl -- get nodes



minikube kubectl -- create namespace argocd

minikube kubectl -- apply --server-side -n argocd -f "https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml"

minikube kubectl -- -n argocd rollout status deployment/argocd-server --timeout=5m



minikube kubectl -- apply -f bootstrap/podinfo.yaml

minikube kubectl -- apply -f bootstrap/monitoring.yaml

```



Attendre que Prometheus et Grafana soient prêts et que les deux CRD

soient disponibles, puis charger la configuration de collecte.



```powershell

minikube kubectl -- -n monitoring get pods

minikube kubectl -- get crd servicemonitors.monitoring.coreos.com prometheusrules.monitoring.coreos.com

minikube kubectl -- apply -f bootstrap/monitoring-config.yaml

```



Ouvrir un terminal PowerShell par interface et garder chaque port-forward actif.



| Interface | Commande | Adresse |

|---|---|---|

| Argo CD | `minikube kubectl -- -n argocd port-forward svc/argocd-server 8080:443` | `https://localhost:8080` |

| Prometheus | `minikube kubectl -- -n monitoring port-forward svc/prometheus-operated 9090:9090` | `http://localhost:9090` |

| Grafana | `minikube kubectl -- -n monitoring port-forward svc/monitoring-grafana 3000:80` | `http://localhost:3000` |



Dans Grafana, importer `dashboard/tp4-gitops.json` et sélectionner la source

Prometheus si Grafana demande de l'associer au dashboard.



\## Requêtes PromQL



| Panneau | Requête |

|---|---|

| Répliques désirées | `kube\_deployment\_spec\_replicas{namespace="demo",deployment="podinfo"}` |

| Répliques disponibles | `kube\_deployment\_status\_replicas\_available{namespace="demo",deployment="podinfo"}` |

| Répliques indisponibles | `kube\_deployment\_spec\_replicas{namespace="demo",deployment="podinfo"} - kube\_deployment\_status\_replicas\_available{namespace="demo",deployment="podinfo"}` |

| État Argo CD | `argocd\_app\_info{name="podinfo"}` |

| Synchronisations sur 5 min | `sum by (phase) (increase(argocd\_app\_sync\_total{name="podinfo"}\[5m]))` |

| CPU de Podinfo | `sum(rate(container\_cpu\_usage\_seconds\_total{namespace="demo",container="podinfo"}\[2m]))` |

| Mémoire de Podinfo | `sum(container\_memory\_working\_set\_bytes{namespace="demo",container="podinfo"})` |

| Redémarrages sur 5 min | `sum(increase(kube\_pod\_container\_status\_restarts\_total{namespace="demo",pod=\~"podinfo-.\*"}\[5m]))` |

| État de l'alerte | `ALERTS{alertname="PodinfoImagePullFailure"}` |



Prometheus déclenche `PodinfoImagePullFailure` si la condition

`ErrImagePull` ou `ImagePullBackOff` reste détectée pendant deux minutes.



\## Démonstrations



1\. \*\*Déploiement automatique.\*\* Avec une réplique au départ, passer

&#x20;  `count` à `2` dans l'overlay, puis faire un commit et un push.

&#x20;  Vérifier qu'Argo CD applique le changement et que le Deployment atteint

&#x20;  `2/2`.

2\. \*\*Self-healing.\*\* Avec deux répliques déclarées dans Git, exécuter

&#x20;  `minikube kubectl -- -n demo scale deployment podinfo --replicas=1`.

&#x20;  Observer le retour automatique à `2/2`.

3\. \*\*Image invalide.\*\* Pousser le tag `tp4-tag-inexistant` et relever le SHA

&#x20;  du commit. Observer `ImagePullBackOff` dans Kubernetes et

&#x20;  `Synced / Degraded` dans Argo CD.

4\. \*\*Rollback Argo CD.\*\* Désactiver temporairement l'Auto-Sync, puis

&#x20;  sélectionner une révision saine dans \*\*History and Rollback\*\*.

&#x20;  Vérifier que les deux pods sont disponibles. Annuler ensuite le commit

&#x20;  fautif avec `git revert`, pousser le correctif et réactiver l'Auto-Sync.

5\. \*\*Observabilité et rétablissement via Git.\*\* Avec l'Auto-Sync actif,

&#x20;  rejouer la panne et observer la baisse des répliques disponibles dans

&#x20;  Grafana. Attendre `Firing` dans Prometheus, puis annuler ce nouveau commit

&#x20;  avec `git revert` et pousser. Vérifier le retour à `Synced / Healthy`,

&#x20;  à deux répliques disponibles et à une alerte `Inactive`.

