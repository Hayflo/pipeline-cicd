# Automated Virtual Infrastructure Deployment Pipeline

Pipeline CI/CD automatisé avec **Jenkins**, **Ansible** et **VMware Workstation**, déployant une infrastructure web + base de données + monitoring sur des VMs locales.

---

## Stack technique

| Outil | Rôle |
|---|---|
| Jenkins | Orchestration du pipeline CI/CD |
| Ansible | Provisioning & déploiement sur les VMs |
| Git / GitHub | Versioning & déclenchement du pipeline (Poll SCM) |
| VMware Workstation | Virtualisation locale |
| Nginx | Serveur web (web-server) |
| MariaDB | Base de données (db-server) |
| Graylog 5 | Centralisation et visualisation des logs |
| OpenSearch 2 | Moteur d'indexation des logs (dépendance Graylog) |
| MongoDB 6 | Stockage de la configuration Graylog |
| Filebeat | Agent de collecte de logs (Nginx + MariaDB) |

---

## Architecture

```
GitHub (repo)
     │
     │  Poll SCM
     ▼
Jenkins Server  ──────────────────────────────────────────┐
192.168.56.10                                              │
     │                                                     │
     │  playbook.yml + graylog.yml                         │
     ├──────────────────────►  Web Server                  │
     │                          192.168.56.11              │
     │                          Nginx                      │
     │                          Filebeat ──────────────┐   │
     │                                                  │   │
     └──────────────────────►  DB Server                │   │
                                192.168.56.12           │   │
                                MariaDB                 │   │
                                Filebeat ───────────┐   │   │
                                MongoDB             │   │   │
                                OpenSearch          │   │   │
                                Graylog 5 ◄─────────┘───┘   │
                                  UI :9000                   │
                                  Beats input :5044          │
```

---

## Structure du projet

```
.
├── Jenkinsfile
└── ansible/
    ├── hosts
    ├── playbook.yml
    ├── rollback.yml
    └── graylog.yml
```

---

## Inventaire Ansible — `ansible/hosts`

Deux groupes de serveurs cibles :

| Groupe | IP | Rôle |
|---|---|---|
| `webservers` | `192.168.56.11` | Nginx |
| `dbservers` | `192.168.56.12` | MariaDB + Graylog |

- Utilisateur SSH : `hayflo`
- Clé SSH : `/var/lib/jenkins/.ssh/id_ed25519`
- `StrictHostKeyChecking` désactivé pour les VMs locales

---

## Playbook de déploiement — `ansible/playbook.yml`

### Play 1 — Web Server (`webservers`)

1. Mise à jour des paquets APT
2. Installation de **Nginx**
3. Démarrage et activation du service
4. Création du dossier `/var/www/html`
5. Déploiement d'une page `index.html` (`<h1>Deploiement reussi</h1>`)

### Play 2 — DB Server (`dbservers`)

1. Mise à jour des paquets APT
2. Installation de **MariaDB**
3. Démarrage et activation du service

---

## Playbook Graylog — `ansible/graylog.yml`

Déploiement de la stack de monitoring centralisée en 3 plays.

### Play 1 — Stack Graylog sur le db-server (`192.168.56.12`)

Installation dans l'ordre suivant :

1. **MongoDB 6** — stockage de la configuration Graylog
2. **OpenSearch 2** — indexation des logs (mode `single-node`, sécurité désactivée)
3. **Graylog 5** — interface de visualisation et d'ingestion des logs
   - UI accessible sur le port `9000`
   - Réception Filebeat sur le port `5044` (Beats input)

### Play 2 — Filebeat sur le web-server (`192.168.56.11`)

Collecte et envoi des logs Nginx vers Graylog :
- `/var/log/nginx/access.log`
- `/var/log/nginx/error.log`

### Play 3 — Filebeat sur le db-server (`192.168.56.12`)

Collecte et envoi des logs MariaDB vers Graylog :
- `/var/log/mysql/mysql.log` (logs généraux activés)
- `/var/log/mysql/error.log`

> **Action manuelle requise après le premier déploiement** : créer le Beats Input dans l'UI Graylog → System → Inputs → Beats → port `5044`.

---

## Playbook de rollback — `ansible/rollback.yml`

Déclenché automatiquement par Jenkins en cas d'échec du pipeline.

### Rollback Web Server

- Réinstallation de Nginx si absent
- Remise en place de la page par défaut (`<h1>Rollback effectué</h1>`)
- Redémarrage du service

### Rollback DB Server

- Réinstallation de MariaDB si absente
- Redémarrage du service

---

## Pipeline Jenkins — `Jenkinsfile`

### Stages

```
Checkout → Vérification Ansible → Test connectivité VMs → Déploiement Ansible → Déploiement Graylog → Vérification déploiement
```

| Stage | Description |
|---|---|
| **Checkout** | Clone le repo, affiche la branche et le commit |
| **Vérification Ansible** | Vérifie la version d'Ansible + syntax-check de `playbook.yml` et `graylog.yml` |
| **Test connectivité VMs** | Ping Ansible sur tous les hôtes |
| **Déploiement Ansible** | Exécute `playbook.yml` (Nginx + MariaDB) |
| **Déploiement Graylog** | Exécute `graylog.yml` (Graylog + Filebeat sur les deux VMs) |
| **Vérification déploiement** | `curl` sur le web-server + vérification de l'API Graylog |

### Post-actions

| Condition | Action |
|---|---|
| `success` | Log de confirmation + URL de l'UI Graylog |
| `failure` | Lancement automatique de `rollback.yml` |
| `always` | Log du statut final du build |

---

## Variables d'environnement Jenkins

| Variable | Valeur |
|---|---|
| `WEB_SERVER` | `192.168.56.11` |
| `DB_SERVER` | `192.168.56.12` |
| `INVENTORY` | `ansible/hosts` |
| `PLAYBOOK` | `ansible/playbook.yml` |
| `ROLLBACK` | `ansible/rollback.yml` |
| `GRAYLOG` | `ansible/graylog.yml` |
| `ANSIBLE_HOST_KEY_CHECKING` | `False` |

---

## Déclenchement

Le pipeline est déclenché via **Poll SCM** — Jenkins surveille le dépôt GitHub à intervalle régulier et lance automatiquement un build lors d'un nouveau commit.

---

## Accès aux interfaces

| Service | URL | Identifiants par défaut |
|---|---|---|
| Jenkins | `http://192.168.56.10:8080` | configurés à l'installation |
| Graylog UI | `http://192.168.56.12:9000` | `admin` / défini dans `graylog.yml` |

---

## Prérequis

- Jenkins installé sur `192.168.56.10` avec le plugin Git
- Ansible installé sur le Jenkins server (`ansible --version`)
- Clé SSH ed25519 générée et déployée sur les VMs cibles
- VMs accessibles en réseau host-only depuis le Jenkins server
- **db-server : minimum 2 Go de RAM** (Graylog + OpenSearch + MongoDB + MariaDB)

---

## Ordre de démarrage des services (db-server)

En cas de redémarrage de la VM, systemd démarre les services automatiquement dans cet ordre :

```
MongoDB → OpenSearch → Graylog → MariaDB → Filebeat
```
