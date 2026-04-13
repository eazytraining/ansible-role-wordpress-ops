# ansible-role-wordpress-ops

> Rôle Ansible réutilisable – Routines opérationnelles d'un WordPress dockerisé.

Ce repo contient **uniquement le rôle**. Il est conçu pour être appelé
depuis un repo playbook séparé via `requirements.yml`.

## Collections requises

```bash
ansible-galaxy collection install community.docker amazon.aws
```

## Variables

Voir [`defaults/main.yml`](defaults/main.yml) pour la liste complète et documentée.

### Variables S3 clés

| Variable | Défaut | Description |
|----------|--------|-------------|
| `s3_enabled` | `false` | Activer l'upload S3 |
| `s3_bucket` | `mon-bucket-wordpress-backup` | Nom du bucket |
| `s3_region` | `eu-west-1` | Région AWS |
| `s3_prefix` | `wordpress` | Préfixe des clés S3 |
| `aws_access_key` | `""` | Laisser vide si IAM role EC2 |
| `aws_secret_key` | `""` | Laisser vide si IAM role EC2 |

## Tags disponibles

| Tag | Routine |
|-----|---------|
| `backup_db` | Backup de la base de données (+ upload S3 optionnel) |
| `backup_files` | Backup fichiers + DB via handler (+ upload S3 optionnel) |
| `restore_db` | Restauration de la base de données |
| `restore_files` | Restauration fichiers + DB via handler |
| `update_plugins` | Mise à jour de tous les plugins WP |
| `manage_plugin` | Activation ou désactivation d'un plugin |
| `cleanup` | Suppression des backups locaux expirés |


# Validation du Rôle WordPress avec Molecule

> Lab EAZYTraining — Ansible + Molecule (Docker) pour standardiser les tests du rôle `wordpress_ops` en préproduction.

---

## Architecture de test

```
┌─────────────────────────────────────────────────────────────────┐
│  Host (EC2 / machine locale)                                    │
│  Docker daemon                                                  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  molecule-wp-host  (instance Molecule)                   │   │
│  │  Ubuntu 22.04 + Docker CLI + Ansible                     │   │
│  │  /var/run/docker.sock monté depuis le host               │   │
│  │                                                          │   │
│  │  ansible-playbook converge.yml  ──► role wordpress_ops   │   │
│  │         │                                                │   │
│  │         │ community.docker.docker_container_exec         │   │
│  │         ▼                                                │   │
│  │  ┌────────────────┐    ┌────────────────┐               │   │
│  │  │ molecule-       │    │ molecule-db-1  │               │   │
│  │  │ wordpress-1     │◄──►│ mysql:5.7      │               │   │
│  │  │ conetix/wp-cli  │    │                │               │   │
│  │  └────────────────┘    └────────────────┘               │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Prérequis

```bash
# Python 3.10+
python3 --version

# Docker
docker info

# Installation avec pipx (recommandé)
sudo apt install pipx
pipx install molecule==26.3.0
pipx inject molecule molecule-docker==2.1.0
pipx inject molecule docker==7.1.0

# OU avec pip
pip install -r requirements.txt

```

# charger le path de molecule
export PATH="$HOME/.local/bin:$PATH"
source ~/.bashrc
molecule --version

# Ensuite injecter le plugin Docker qui manque encore :
pipx inject molecule molecule-docker
pipx inject molecule docker

# Puis initialiser le scénario :
cd ansible-role-wordpress-ops
molecule init scenario

---

## Structure Molecule

```
molecule/
└── default/
    ├── molecule.yml     ← driver, platforms, séquence de test
    ├── Dockerfile.j2    ← image Ubuntu + Docker CLI + Ansible
    ├── prepare.yml      ← démarre wordpress + mysql
    ├── converge.yml     ← applique le rôle (toutes les routines)
    ├── verify.yml       ← assertions par routine
    └── destroy.yml      ← nettoyage propre
```

---

## Cycle complet de test

```
lint → destroy → syntax → create → prepare → converge → idempotence → verify → destroy
```

### Étape par étape

| Étape | Commandeq | Description |
|-------|----------|-------------|
| Lint | `molecule lint` | yamllint + ansible-lint |
| Syntax | `molecule syntax` | Vérification syntaxe Ansible |
| Create | `molecule create` | Démarrage de l'instance |
| Prepare | `molecule prepare` | Stack WordPress (wp + mysql) |
| Converge | `molecule converge` | Application du rôle |
| Idempotence | `molecule idempotence` | Re-run → 0 `changed` attendu |
| Verify | `molecule verify` | Assertions (SQL, tar, WP-CLI) |
| Destroy | `molecule destroy` | Nettoyage |
| **Full** | **`molecule test`** | **Tout d'un coup** |

---

## Commandes utiles

# Collections Ansible
/home/ubuntu/.local/share/pipx/venvs/molecule/bin/ansible-galaxy \
  collection install -r requirements.yml

```bash
# Cycle complet
molecule test

# Développement itératif (sans destroy)
molecule create
molecule prepare
molecule converge
molecule verify

# Rejouer uniquement le converge (après une modif du rôle)
molecule converge

# Vérifier l'idempotence manuellement
molecule idempotence

# Ouvrir un shell dans l'instance Molecule
molecule login

# Lancer uniquement le lint
molecule lint

# Nettoyer sans tester
molecule destroy

# Mode verbeux
molecule test -- -vvv

# Tester un tag spécifique uniquement
molecule converge -- --tags backup_db
```

---

## Ce que vérifie `verify.yml`

| Routine | Assertion | Outil |
|---------|-----------|-------|
| `backup_db` | Fichier `.sql` présent et non vide | `find` + `head` |
| `backup_db` | En-tête SQL valide (`mysqldump`) | `head -5` |
| `backup_files` | Archive `.tar.gz` présente | `find` |
| `backup_files` | Archive lisible et non corrompue | `tar -tzf` |
| `backup_files` | Contient des fichiers WordPress | `tar -tzf` |
| `restore_db` | Tables `wp_*` présentes | `SHOW TABLES` |
| `restore_db` | Table `wp_options` présente | `SHOW TABLES LIKE` |
| `restore_files` | WordPress répond via WP-CLI | `wp option get siteurl` |
| `uninstall_plugin` | Plugin absent après suppression | `wp plugin is-installed` |

---

## Idempotence — règle d'or

Après un premier `converge`, un second run doit retourner **0 tâche `changed`**.

```bash
molecule idempotence
```

Résultat attendu :
```
PLAY RECAP *****
molecule-wp-host : ok=XX  changed=0  failed=0
```

Si `changed > 0` → le rôle n'est pas idempotent → corriger les tâches concernées.

---

## CI/CD — GitHub Actions

Le workflow `.github/workflows/molecule.yml` se déclenche sur :
- Tout push sur `main` ou `develop` touchant `tasks/`, `handlers/`, `defaults/`, `vars/`, `molecule/`
- Toute Pull Request vers `main`

```
push → lint → molecule test → notify
```

### Badge à ajouter dans le README du rôle

```markdown
![Molecule CI](https://github.com/votre-org/ansible-role-wordpress-ops/actions/workflows/molecule.yml/badge.svg)
```

---

## Placement dans le repo du rôle

Ce dossier `molecule/` se place **à la racine du repo `ansible-role-wordpress-ops`** :

```
ansible-role-wordpress-ops/
├── defaults/main.yml
├── handlers/main.yml
├── tasks/
│   ├── main.yml
│   ├── backup_db.yml
│   └── ...
├── molecule/
│   └── default/
│       ├── molecule.yml
│       ├── Dockerfile.j2
│       ├── prepare.yml
│       ├── converge.yml
│       ├── verify.yml
│       └── destroy.yml
├── requirements.yml
└── .github/
    └── workflows/
        └── molecule.yml
```
