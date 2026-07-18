# Projet MGGT1103 — Adaptation solo (1 laptop, 3 VMs simulées)

## ⚠️ Adaptation à valider avec Dr. NIBITANGA
Le sujet impose 3 laptops physiques en réseau Wi-Fi partagé. En solo, on simule
les 3 nœuds avec 3 VMs VirtualBox sur un réseau **host-only** (192.168.56.0/24)
au lieu du Wi-Fi partagé (192.168.43.0/24). Le comportement (tolérance de
panne, SSO, auto-healing) reste identique et démontrable en soutenance —
il faut juste le mentionner clairement dans le rapport et à l'oral.

## Phase 0 : Installer les outils sur ton Ubuntu

```bash
sudo apt update
sudo apt install -y virtualbox vagrant ansible

# Générer une clé SSH si tu n'en as pas déjà une (le Vagrantfile s'en sert)
ls ~/.ssh/id_rsa.pub || ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""

# Vérifier
vagrant --version
ansible --version
VBoxManage --version
```

## Phase 1 : Lever les 3 VMs

```bash
cd infra/
vagrant up
# ça télécharge la box Ubuntu et crée master, worker1, worker2
# (compte 10-20 min la première fois selon ta connexion)

vagrant status   # les 3 VMs doivent être "running"
```

## Phase 1 (suite) : Hardening + Docker + K3s via Ansible

```bash
cd ../ansible/
ansible -i inventory.ini k3s_cluster -m ping   # doit répondre "pong" x3

ansible-playbook -i inventory.ini setup-cluster.yml
```

## Vérification Phase 1

```bash
vagrant ssh master
kubectl get nodes
# tu dois voir master, worker1, worker2 avec le statut Ready
exit
```

## Prochaines étapes (à faire une fois la Phase 1 validée)
- **Phase 2** : Dockerfile de personnalisation pour Nginx (COPY index.html) +
  docker-compose.yml pour tester Kanboard/Nginx en local
- **Phase 3** : Pipeline GitHub Actions (Hadolint + Trivy + push Docker Hub)
- **Phase 4** : Manifests K8s (deployment/service/ingress) + Keycloak SSO
  avec Traefik ForwardAuth
- **Phase 5** : Stack Prometheus/Loki/Grafana

Dis-moi quand la Phase 1 tourne (les 3 nœuds `Ready`) et on attaque la Phase 2.
