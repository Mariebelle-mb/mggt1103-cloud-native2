# MGGT1103 — Conception et Automatisation d'une Plateforme Cloud-Native Distribuée

**Variante 3 : Télécommunications (Support et Tickets)**
Université Espoir d'Afrique — Dr. NIBITANGA Romeo
Faculté de Génie et Gestion de la Télécommunication (GGT)

---

## 🎯 Aperçu du projet

Ce dépôt contient l'infrastructure complète d'une plateforme cloud-native distribuée pour un
service de support télécom, comprenant :

- **App 1 — Kanboard** : gestionnaire de tâches/tickets d'intervention.
- **App 2 — Portail Nginx personnalisé** : page d'accueil client statique, personnalisée via
  Dockerfile local.

L'ensemble est orchestré par **K3s (Kubernetes)**, exposé via **Traefik**, sécurisé par un
**Single Sign-On (SSO) Keycloak**, observé par la stack **Prometheus / Loki / Grafana**, et
déployé par un pipeline **CI/CD GitHub Actions** (Hadolint + Trivy + Docker Hub).

---

## 🏗️ Architecture

```
                    Wi-Fi / Réseau local (VirtualBox - réseau privé)
        ┌─────────────────────────────────────────────────────────┐
        │  VM 1 : MASTER          VM 2 : WORKER1     VM 3 : WORKER2 │
        │  192.168.56.10          192.168.56.20      192.168.56.30  │
        │  - Control Plane K3s    - Agent K3s         - Agent K3s   │
        │  - Traefik (Ingress)    - Pods Kanboard      - Pods Nginx │
        │  - Keycloak (SSO)       - Pods Nginx         - Pods Nginx │
        │  - Prometheus/Loki      - oauth2-proxy                    │
        │  - Grafana                                                │
        └─────────────────────────────────────────────────────────┘
```

> ⚠️ **Limitation connue** : faute d'accès simultané à 3 laptops physiques distincts durant les
> séances de travail, l'infrastructure a été déployée sur **une seule machine hôte** via 3 VMs
> Vagrant/VirtualBox en réseau privé, simulant fidèlement une architecture multi-nœuds K3s
> (1 master + 2 workers). Tous les mécanismes de résilience, SSO, CI/CD et observabilité ont
> été validés dans cette configuration. Le passage à un réseau bridged multi-laptops (`Wi-Fi
> partagé, 192.168.43.0/24`) reste possible en modifiant le mode réseau du `Vagrantfile` et
> l'inventaire Ansible.

---

## 📁 Structure du dépôt

```
.
├── ansible/                  # Durcissement + bootstrap K3s
│   ├── inventory.ini         # Inventaire des 3 nœuds
│   ├── setup-cluster.yml     # Playbook (SSH hardening, UFW, Docker, K3s)
│   └── secrets.yml           # Secrets chiffrés (Ansible Vault)
├── app/
│   ├── docker-compose.yml    # Test local des 2 applications
│   └── nginx-portal/         # Dockerfile + index.html personnalisé
├── .github/workflows/
│   └── cicd.yml              # Pipeline : Hadolint → Build → Trivy → Docker Hub
├── k8s/
│   ├── base/                 # Kanboard, Nginx, Ingress principal
│   ├── oauth2-proxy/         # Middleware Traefik + Ingress SSO
│   └── keycloak/             # Déploiement Keycloak
├── infra/                    # Vagrantfile + copie miroir de k8s/ (montée dans les VMs)
└── README.md
```

---

## 🚀 Déploiement complet (depuis zéro)

### 1. Prérequis
- VirtualBox + Vagrant installés
- Ansible installé (`sudo apt install ansible`)
- Helm installé sur la VM master (voir étape 4)

### 2. Lever les machines virtuelles

```bash
cd infra
vagrant up
```

Vérification : 3 VMs `master`, `worker1`, `worker2` doivent démarrer.

### 3. Durcissement système + installation K3s (Ansible + Vault)

```bash
cd ansible
ansible-playbook -i inventory.ini setup-cluster.yml --ask-vault-pass
```

Le mot de passe du coffre-fort Vault est détenu par l'équipe (non versionné en clair).

Vérification :
```bash
vagrant ssh master -c "sudo kubectl get nodes"
```
→ Les 3 nœuds doivent apparaître avec le statut `Ready`.

### 4. Déployer les applications, Keycloak, Traefik et le SSO

```bash
cd infra
vagrant ssh master -c "sudo kubectl apply -f /vagrant/k8s/base/"
vagrant ssh master -c "sudo kubectl apply -f /vagrant/k8s/keycloak/"
vagrant ssh master -c "sudo kubectl apply -f /vagrant/k8s/oauth2-proxy/"
```

### 5. Déployer l'observabilité (Helm)

```bash
vagrant ssh master -c "curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash"
vagrant ssh master -c "sudo helm repo add prometheus-community https://prometheus-community.github.io/helm-charts"
vagrant ssh master -c "sudo helm repo add grafana https://grafana.github.io/helm-charts"
vagrant ssh master -c "sudo helm repo update"

vagrant ssh master -c "sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm install monitoring prometheus-community/kube-prometheus-stack -n mggt1103-projet --set grafana.adminPassword=Ggt2026!"

vagrant ssh master -c "sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm install loki grafana/loki-stack -n mggt1103-projet --set promtail.enabled=true --set grafana.enabled=false --set loki.image.tag=2.9.10 --set loki.isDefault=false"
```

### 6. Résolution DNS locale

Sur la machine de présentation, ajoutez à `/etc/hosts` (Linux/Mac) ou
`C:\Windows\System32\drivers\etc\hosts` (Windows) :

```
192.168.56.10   auth.projet.local
192.168.56.10   kanboard.projet.local
192.168.56.10   portail.projet.local
192.168.56.10   grafana.projet.local
```

---

## 🔑 Accès et identifiants de test

| Service | URL | Identifiants |
|---|---|---|
| Kanboard (SSO) | http://kanboard.projet.local | `agent1` / `Test1234!` (via Keycloak) |
| Portail client | http://portail.projet.local | Accès automatique après SSO Kanboard |
| Grafana | http://grafana.projet.local | `admin` / `Ggt2026!` |
| Keycloak Admin | http://auth.projet.local | Voir secrets Ansible Vault |

---

## ✅ Preuves de fonctionnement (démontrées en séance)

- **SSO** : connexion à `kanboard.projet.local` via Keycloak, puis accès immédiat à
  `portail.projet.local` sans ressaisir le mot de passe.
- **Auto-healing** : suppression manuelle du pod Kanboard (`kubectl delete pod ...`) →
  recréation automatique en quelques secondes par Kubernetes.
- **CI/CD** : chaque `git push` sur `app/nginx-portal/**` déclenche Hadolint (lint du
  Dockerfile), un build Docker, un scan Trivy (CVE), puis une publication sur Docker Hub.
- **Observabilité** : dashboard Grafana "Node Exporter Full" affichant l'usage CPU/mémoire/disque
  des 3 nœuds, avec un panneau de logs Loki en temps réel (`{namespace="mggt1103-projet"}`).

---

## 🛠️ Choix techniques notables

- **Kanboard reste à `replicas: 1`** (contrainte technique : son volume persistant est en mode
  `ReadWriteOnce`, incompatible avec plusieurs réplicas simultanés sans stockage partagé
  réseau). Le portail Nginx (stateless) est à `replicas: 3`.
- **oauth2-proxy** protège les 2 applications via un middleware Traefik `ForwardAuth`
  (`authResponseHeaders: X-Auth-Request-User`), avec `--set-xauthrequest=true` et
  `--cookie-domain=.projet.local` pour permettre le SSO entre sous-domaines.
- **Kanboard** est configuré en mode reverse-proxy-auth (`REVERSE_PROXY_AUTH`,
  `REVERSE_PROXY_USER_HEADER`, `TRUSTED_PROXY_NETWORKS`) pour accepter l'identité transmise
  par oauth2-proxy sans formulaire de connexion natif.

---

## 👥 Équipe

| Membre | Matricule | Pôle technique |
|---|---|---|
| MARIAM COBWA Marybella | 26200308 | Orchestration & Identité (K8s, Traefik, SSO Keycloak), CI/CD, Observabilité |
| MUGISHA WILSON | 26200159 | Cloud-Native & CI/CD (Dockerfiles, GitHub Actions, Hadolint/Trivy) |
| MUGISHO BINTU Pascal | 26200160 | Infrastructure & SecOps (Vagrant, Ansible, Ansible Vault, cluster physique bridgé) |
| MUNIHIRE NYANGUBA Fidèle | 26200232 | Observabilité (SRE) — stack PLG (Prometheus, Loki, Grafana) |

Encadré par : Dr. NIBITANGA Romeo — Faculté d'Ingénierie et Technologie, Département de
Génie et Gestion de Télécommunication, Université Espoir d'Afrique.
