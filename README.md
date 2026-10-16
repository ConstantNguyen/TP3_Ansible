# TP3 - Ansible

## 3-1 Commandes d'inventaire et de base

### Inventaire : `ansible/inventories/setup.yml`

```yaml
all:
 vars:
   ansible_user: admin
   ansible_ssh_private_key_file: /home/Constant/.ssh/id_rsa
 children:
   prod:
     hosts: constant.nguyen.takima.school
```

- `all` : groupe qui contient tous les hôtes.
- `vars` : variables appliquées à tous les hôtes. `ansible_user` est l'utilisateur SSH et `ansible_ssh_private_key_file` la clé privée utilisée pour se connecter.
- `children` / `prod` / `hosts` : sous-groupe `prod` qui contient le serveur géré.

### Commandes

```bash
ansible all -i inventories/setup.yml -m ping
```
Teste la connexion SSH et la présence de Python sur le serveur. Le module `ping` n'est pas un ping réseau : la réponse `pong` confirme qu'Ansible peut exécuter ses modules sur la machine.

```bash
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
```
Le module `setup` collecte les faits (variables découvertes sur la machine, préfixées par `ansible_`). Le filtre ne garde que ceux qui concernent la distribution (nom, version, nom de code).

```bash
ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=absent" --become
```
Le module `apt` décrit l'état voulu : `apache2` doit être absent. `--become` exécute la commande en root. La commande est idempotente : si Apache est déjà désinstallé, Ansible ne change rien.

```bash
ansible all -i inventories/setup.yml -m command -a "docker --version" --become
```
Le module `command` exécute une commande sur le serveur distant. Elle sert ici à vérifier que Docker est installé.

```bash
ansible-playbook -i inventories/setup.yml playbook.yml --syntax-check
```
Vérifie la syntaxe du playbook sans l'exécuter.

```bash
ansible-playbook -i inventories/setup.yml playbook.yml --ask-vault-pass
```
Exécute le playbook. `--ask-vault-pass` demande le mot de passe du vault qui contient les données privées.

## 3-2 Playbook

### `ansible/playbook.yml`

```yaml
- hosts: all
  gather_facts: true
  become: true

  roles:
    - docker
    - network
    - database
    - app
    - front
    - proxy
```

- `hosts: all` : le playbook s'applique à tous les hôtes de l'inventaire.
- `gather_facts: true` : collecte les faits avant les tâches. Le rôle `docker` utilise `ansible_facts['distribution_release']` pour construire l'adresse du dépôt apt de Docker.
- `become: true` : toutes les tâches s'exécutent en root.
- `roles` : liste des rôles, exécutés dans l'ordre. Docker doit être installé avant tout, le réseau doit exister avant les conteneurs, la base et l'API avant le front, et le proxy vient en dernier car il redirige vers eux.

### Rôle `docker` : `roles/docker/tasks/main.yml`

| Tâche | Module | Rôle |
|---|---|---|
| Install required packages | `apt` | Installe les prérequis pour ajouter un dépôt apt en HTTPS (`apt-transport-https`, `ca-certificates`, `curl`, `gnupg`, `lsb-release`) et `python3-venv` |
| Add Docker GPG key | `apt_key` | Ajoute la clé publique de Docker pour vérifier l'authenticité des paquets |
| Add Docker APT repository | `apt_repository` | Ajoute le dépôt officiel Docker, adapté à la version de Debian grâce au fait `distribution_release` |
| Install Docker | `apt` | Installe `docker-ce` |
| Install Python3 and pip3 | `apt` | Installe `python3` et `python3-pip` |
| Create a virtual environment | `command` | Crée `/opt/docker_venv`. `creates` évite de la recréer si elle existe déjà |
| Install Docker SDK for Python | `command` | Installe la librairie Python `docker` dans ce venv, nécessaire aux modules `docker_network`, `docker_container` et `docker_login` |
| Make sure Docker is running | `service` | S'assure que le service `docker` est démarré |
| Login to DockerHub | `docker_login` | Connecte le serveur à DockerHub pour télécharger les images. Le token vient du vault |

### Rôle `network`

Crée le réseau Docker `app-network` avec `docker_network`. Sur un réseau créé par l'utilisateur, les conteneurs se trouvent par leur nom (par exemple `tp-db`).

### Rôles `database`, `app`, `front`, `proxy`

Chacun lance un conteneur. Ils sont détaillés dans la section suivante.

### `ansible_python_interpreter`

Les modules `docker_*` ont besoin de la librairie Python `docker`, installée dans le venv `/opt/docker_venv` et non dans le Python du système. Chaque tâche qui utilise un module Docker précise donc `ansible_python_interpreter: /opt/docker_venv/bin/python`.

### Ansible Vault

Les données privées (token DockerHub, utilisateur et mot de passe de la base) sont dans `group_vars/all/vault.yml`, chiffré avec `ansible-vault`. Ansible charge automatiquement `group_vars/all/`. Les rôles utilisent les variables `{{ dockerhub_token }}`, `{{ postgres_user }}` et `{{ postgres_password }}`.

## 3-3 Configuration des tâches docker_container

Le module `docker_container` équivaut à un `docker run` : il télécharge l'image, crée le conteneur et le démarre. Il est idempotent : si le conteneur existe déjà avec la même configuration, rien ne change.

### Base de données : `roles/database/tasks/main.yml`

```yaml
- name: Launch database
  community.docker.docker_container:
    name: tp-db
    image: constant4n/tp-devops-database
    networks:
      - name: app-network
    env:
      POSTGRES_DB: db
      POSTGRES_USER: "{{ postgres_user }}"
      POSTGRES_PASSWORD: "{{ postgres_password }}"
  vars:
    ansible_python_interpreter: /opt/docker_venv/bin/python
```

- `name: tp-db` : nom du conteneur, qui est aussi son nom sur le réseau.
- `image` : image DockerHub de la base.
- `networks` : réseau `app-network`.
- `env` : variables d'environnement lues par l'image PostgreSQL pour créer la base, l'utilisateur et le mot de passe.
- Pas de `ports` : la base n'est pas accessible depuis l'extérieur.

### API : `roles/app/tasks/main.yml`

```yaml
- name: Launch app
  community.docker.docker_container:
    name: simple-api
    image: constant4n/tp-devops-simple-api
    pull: always
    networks:
      - name: app-network
    env:
      SPRING_DATASOURCE_URL: jdbc:postgresql://tp-db:5432/db
      SPRING_DATASOURCE_USERNAME: "{{ postgres_user }}"
      SPRING_DATASOURCE_PASSWORD: "{{ postgres_password }}"
  vars:
    ansible_python_interpreter: /opt/docker_venv/bin/python
```

- `name: simple-api` : nom attendu par la configuration du proxy.
- `env` : variables `SPRING_DATASOURCE_*` qui remplacent les valeurs de `application.yml`. L'URL utilise `tp-db`, le nom du conteneur de la base. Les identifiants sont identiques à ceux de la base.
- Pas de `ports` : l'API n'est joignable que par le proxy.

### Front : `roles/front/tasks/main.yml`

```yaml
- name: Launch front
  community.docker.docker_container:
    name: tp-www
    image: constant4n/tp-devops-front
    pull: always
    networks:
      - name: app-network
  vars:
    ansible_python_interpreter: /opt/docker_venv/bin/python
```

- `name: tp-www` : nom utilisé par le proxy pour joindre le front.
- Pas de `ports` : le front n'est joignable que par le proxy.

### Proxy : `roles/proxy/tasks/main.yml`

```yaml
- name: Launch proxy
  community.docker.docker_container:
    name: tp-http-server
    image: constant4n/tp-devops-httpd
    pull: always
    ports:
      - "80:80"
    networks:
      - name: app-network
  vars:
    ansible_python_interpreter: /opt/docker_venv/bin/python
```

- `ports: "80:80"` : seul conteneur exposé. Le port 80 du serveur est redirigé vers le port 80 du conteneur.
- La configuration Apache (`my-httpd.conf`) est embarquée dans l'image : `/` est transmis à `tp-www` et `/api/` à `simple-api:8080`.

### Paramètres communs

- `pull: always` : vérifie DockerHub à chaque exécution et recrée le conteneur si l'image a changé. Sans lui, Ansible réutilise l'image déjà présente sur le serveur.
- `networks` : tous les conteneurs sont sur le même réseau pour se joindre par leur nom.
- Chaque nom écrit dans une configuration (proxy, URL JDBC) doit correspondre exactement au `name` du conteneur visé.

## Continuous Deployment : est-il sûr de déployer automatiquement chaque nouvelle image du hub ?

**Non, ce n'est pas sûr.**

Chaque image poussée sur DockerHub part directement en production, sans contrôle :

- Une image cassée ou non testée est déployée et peut mettre l'application hors service.
- Une image compromise (token DockerHub volé, push depuis une mauvaise branche, dépendance malveillante) s'exécute sur le serveur.
- Avec le tag `latest`, on ne sait pas quelle version tourne et il est difficile de revenir en arrière.
- Une image peut contenir des vulnérabilités connues qui ne sont pas détectées.

### Comment le sécuriser

- Ne déployer que si les tests passent et seulement depuis la branche `main`.
- Utiliser des tags de version ou le SHA du commit au lieu de `latest`, pour identifier la version déployée et pouvoir faire un rollback.
- Protéger la branche `main` (pull request et review obligatoires) et ajouter une approbation manuelle avant le déploiement, via un GitHub Environment.
- Scanner les images avec un outil comme Trivy avant de déployer.
- Limiter les droits des tokens (DockerHub, SSH) et ne stocker les données privées que dans les secrets GitHub et dans Ansible Vault.