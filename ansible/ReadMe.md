<!-- ansible/README.md -->
# TP DevOps - Partie 3 : Ansible

Déploiement automatisé de l'application 3-tiers (base de données, API, front, reverse proxy) sur un serveur Debian distant avec Ansible, puis déploiement continu via GitHub Actions.

Application déployée : http://armand.labernardiere.takima.school

## Architecture déployée

```
                    Internet
                       │
                   port 80
                       │
               ┌───────▼───────┐
               │     httpd     │  Apache (reverse proxy)
               └───┬───────┬───┘
            /api/  │       │  /
           ┌───────▼──┐ ┌──▼──────┐
           │simple-api│ │  front  │  Vue.js servi par Nginx
           └────┬─────┘ └─────────┘
                │
           ┌────▼─────┐
           │ database │  PostgreSQL + volume db-volume
           └──────────┘

        Tous les conteneurs sont sur le réseau my-network
```

Seul Apache expose un port sur le serveur. Il redirige `/api/` vers le backend et le reste vers le front.

## Structure

```
ansible/
├── inventories/
│   └── setup.yml          # Serveur cible et connexion SSH
├── playbook.yml           # Playbook principal
└── roles/
    ├── docker/            # Installation de Docker sur le serveur
    ├── network/           # Création du réseau Docker
    ├── database/          # Conteneur PostgreSQL
    ├── app/               # Conteneur de l'API Spring Boot
    ├── front/             # Conteneur du front
    └── proxy/             # Conteneur Apache (reverse proxy)
```

## Prérequis

- Ansible installé sur la machine locale
- Collection `community.docker` (`ansible-galaxy collection install community.docker`)
- Clé SSH privée donnant accès au serveur (user `admin`)
- Images publiées sur Docker Hub (fait automatiquement par la CI)

## Inventaire

`inventories/setup.yml` définit le serveur cible et la connexion SSH :

```yaml
all:
  vars:
    ansible_user: admin
    ansible_ssh_private_key_file: /chemin/vers/id_rsa
    ansible_python_interpreter: /usr/bin/python3
  children:
    prod:
      hosts:
        armand.labernardiere.takima.school:
```

Vérifier la connexion :

```bash
ansible all -i inventories/setup.yml -m ping
```

Récupérer des informations sur le serveur :

```bash
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
```

## Playbook

```yaml
- hosts: all
  gather_facts: true
  become: true

  vars:
    db_password: pwd

  roles:
    - docker
    - network
    - database
    - app
    - front
    - proxy
```

- `gather_facts: true` : récupère les informations du serveur (distribution, version), utilisées par le rôle docker pour choisir le bon dépôt APT
- `become: true` : exécute les tâches en root (installation de paquets, gestion de Docker)
- Les rôles sont joués dans l'ordre : Docker doit être installé avant de créer le réseau, et le réseau doit exister avant de lancer les conteneurs

Lancer le déploiement :

```bash
ansible-playbook -i inventories/setup.yml playbook.yml
```

## Rôles

Les rôles ont été créés avec :

```bash
ansible-galaxy init roles/<nom_du_role>
```

### docker

Installe Docker sur le serveur Debian :

1. Installe les paquets nécessaires (`ca-certificates`, `curl`, `gnupg`, `python3-venv`...)
2. Ajoute la clé GPG et le dépôt APT officiel de Docker
3. Installe `docker-ce`
4. Crée un environnement virtuel Python `/opt/docker_venv` et y installe le SDK Docker pour Python, nécessaire aux modules `community.docker`
5. Vérifie que le service Docker est démarré

Le SDK est installé dans un venv car les versions récentes de Debian refusent les `pip install` système (PEP 668). Les autres rôles utilisent donc `ansible_python_interpreter: /opt/docker_venv/bin/python`.

### network

Crée le réseau Docker `my-network`, qui permet aux conteneurs de se joindre par leur nom :

```yaml
- name: Create Docker network
  community.docker.docker_network:
    name: my-network
    state: present
```

### database

Lance PostgreSQL depuis l'image `enefixx/tp-devops-database`, avec un volume nommé pour persister les données :

```yaml
- name: Run database
  community.docker.docker_container:
    name: database
    image: enefixx/tp-devops-database:latest
    pull: true
    state: started
    restart_policy: always
    env:
      POSTGRES_PASSWORD: "{{ db_password }}"
    volumes:
      - db-volume:/var/lib/postgresql/data
    networks:
      - name: my-network
```

### app

Lance l'API Spring Boot depuis `enefixx/tp-devops-simple-api`. Le backend joint la base par son nom de conteneur (`DATABASE_HOST: database`) :

```yaml
- name: Run backend API
  community.docker.docker_container:
    name: simple-api
    image: enefixx/tp-devops-simple-api:latest
    pull: true
    state: started
    restart_policy: always
    env:
      DATABASE_HOST: database
      DATABASE_PASSWORD: "{{ db_password }}"
    networks:
      - name: my-network
```

### front

Lance le front Vue.js depuis `enefixx/tp-devops-front`. L'image est construite en multistage : Node compile l'application, puis Nginx sert les fichiers statiques. L'URL de l'API est définie dans `front/.env.production` :

```env
VUE_APP_API_URL=armand.labernardiere.takima.school/api
```

### proxy

Lance Apache depuis `enefixx/tp-devops-httpd`. C'est le seul conteneur qui publie un port (`80:80`). Sa configuration route les requêtes :

```apache
<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass /api/ http://simple-api:8080/
ProxyPassReverse /api/ http://simple-api:8080/
ProxyPass / http://front:80/
ProxyPassReverse / http://front:80/
</VirtualHost>
```

L'ordre compte : `/api/` est déclaré avant `/`, sinon toutes les requêtes partiraient vers le front.

### Options communes aux conteneurs

- `pull: true` : télécharge toujours la dernière version de l'image, indispensable pour que le déploiement continu prenne en compte les nouvelles images
- `restart_policy: always` : le conteneur redémarre en cas de crash ou de redémarrage du serveur
- `state: started` : le module est idempotent, relancer le playbook ne recrée un conteneur que si sa configuration ou son image a changé

## Déploiement continu

Le workflow `.github/workflows/main.yml` enchaîne trois jobs à chaque push sur `main` :

1. **test-backend** : build et tests Maven du backend
2. **build-and-push-docker-image** : construit les 4 images (simple-api, database, httpd, front) et les publie sur Docker Hub
3. **deploy** : installe Ansible sur le runner GitHub et lance le playbook sur le serveur

```yaml
deploy:
  needs: build-and-push-docker-image
  if: github.ref == 'refs/heads/main'
  runs-on: ubuntu-24.04
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-python@v5
      with:
        python-version: '3.12'
    - name: Install Ansible
      run: pip install ansible
    - name: Set up SSH key
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
        chmod 600 ~/.ssh/id_rsa
    - name: Run Ansible playbook
      working-directory: ./ansible
      env:
        ANSIBLE_HOST_KEY_CHECKING: "False"
      run: ansible-playbook -i inventories/setup.yml playbook.yml -e ansible_ssh_private_key_file=$HOME/.ssh/id_rsa
```

- La clé SSH est stockée dans le secret `SSH_PRIVATE_KEY` du repo, puis recréée dans un fichier sur le runner
- `-e ansible_ssh_private_key_file=...` remplace le chemin local de l'inventaire par celui du runner
- `ANSIBLE_HOST_KEY_CHECKING=False` évite le blocage à la première connexion SSH (le serveur n'est pas encore dans les `known_hosts` du runner)

Secrets à configurer dans le repo GitHub :

| Secret | Contenu |
|---|---|
| `DOCKERHUB_USERNAME` | Nom d'utilisateur Docker Hub |
| `DOCKERHUB_TOKEN` | Token d'accès Docker Hub |
| `SSH_PRIVATE_KEY` | Clé privée SSH du serveur |

## Pistes d'amélioration

- Chiffrer `db_password` avec Ansible Vault plutôt que de le laisser en clair dans le playbook
- Versionner les images avec des tags (SHA du commit) plutôt que `latest`, pour pouvoir revenir à une version précédente

## Auteur

Armand Labernardière - EFREI, 4e année
