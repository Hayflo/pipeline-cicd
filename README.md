# Automated Virtual Infrastructure Deployment Pipeline

Pipeline CI/CD automatisé avec **Jenkins**, **Ansible** et **VMware Workstation**, déployant une infrastructure web + base de données sur des VMs locales.

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

---

## Architecture

```
GitHub (repo)
     │
     │  Poll SCM
     ▼
Jenkins Server  ──────────────────────────────────┐
192.168.56.10                                      │
     │                                             │
     │  ansible-playbook / rollback.yml            │
     ├──────────────────────►  Web Server          │
     │                          192.168.56.11      │
     │                          Nginx              │
     │                                             │
     └──────────────────────►  DB Server           │
                                192.168.56.12      │
                                MariaDB            │
```

---

## Structure du projet

```
.
├── Jenkinsfile
└── ansible/
    ├── hosts
    ├── playbook.yml
    └── rollback.yml
```

---

## Inventaire Ansible — `ansible/hosts`

Deux groupes de serveurs cibles :

| Groupe | IP | Rôle |
|---|---|---|
| `webservers` | `192.168.56.11` | Nginx |
| `dbservers` | `192.168.56.12` | MariaDB |

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
Checkout → Vérification Ansible → Test connectivité VMs → Déploiement Ansible → Vérification déploiement
```

| Stage | Description |
|---|---|
| **Checkout** | Clone le repo, affiche la branche et le commit |
| **Vérification Ansible** | Vérifie la version d'Ansible + syntax-check du playbook |
| **Test connectivité VMs** | Ping Ansible sur tous les hôtes |
| **Déploiement Ansible** | Exécute `playbook.yml` sur l'inventaire |
| **Vérification déploiement** | `curl` sur le web-server pour valider la réponse HTTP |

### Post-actions

| Condition | Action |
|---|---|
| `success` | Log de confirmation avec la branche |
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
| `ANSIBLE_HOST_KEY_CHECKING` | `False` |

---

## Déclenchement

Le pipeline est déclenché via **Poll SCM** — Jenkins surveille le dépôt GitHub à intervalle régulier et lance automatiquement un build lors d'un nouveau commit.

---

## Prérequis

- Jenkins installé sur `192.168.56.10` avec le plugin Git
- Ansible installé sur le Jenkins server (`ansible --version`)
- Clé SSH ed25519 générée et déployée sur les VMs cibles
- VMs accessibles en réseau host-only depuis le Jenkins server
