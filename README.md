# Docker Swarm – Cluster PCA / PRA

Déploiement d'un cluster Docker Swarm sur des VM Debian pour assurer la continuité (PCA) et la reprise d'activité (PRA) de services critiques.

---

## 1. Architecture

| Nœud   | Rôle                         | IP           |
|--------|------------------------------|--------------|
| Noeud1 | Manager                      | 10.60.8.99   |
| Noeud2 | Manager                      | 10.60.8.100  |
| Noeud3 | Manager + serveur NFS        | 10.60.8.101  |

- **3 managers** : tolérance à la perte d'un manager (quorum Raft 2/3).
- **NFS sur Noeud3** : stockage persistant partagé par tous les nœuds.
- **Réseau overlay `backend`** (attachable) : communication entre les services.

### Services déployés

| Service  | Image                                 | Réplicas | Port  | Volume NFS                  |
|----------|---------------------------------------|----------|-------|-----------------------------|
| registry | registry:2                            | 1        | 5000  | /srv/nfs/swarm/registry     |
| mariadb  | mariadb:11                            | 1        | –     | /srv/nfs/swarm/mariadb      |
| php      | php:8.3-fpm                           | 2        | –     | /srv/nfs/swarm/web          |
| nginx    | nginx:alpine                          | 2        | 80    | /srv/nfs/swarm/web          |
| vscode   | lscr.io/linuxserver/code-server       | 1        | 8443  | /srv/nfs/swarm/vscode + web |

---

## 2. Installation de Docker (sur chaque nœud)

### Nettoyage de l'ancienne installation
```bash
docker swarm leave --force
systemctl stop docker docker.socket containerd
apt purge -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin \
  docker-compose-plugin docker-ce-rootless-extras docker.io containerd runc
apt autoremove -y --purge
umount -l $(mount | grep /var/lib/docker | awk '{print $3}')
rm -rf /var/lib/docker /var/lib/containerd /etc/docker
```

### Installation depuis le dépôt officiel
```bash
apt update
apt install -y ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/debian $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
> /etc/apt/sources.list.d/docker.list

apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker
docker run --rm hello-world
```

---

## 3. Création du cluster Swarm

Sur Noeud1 :
```bash
docker swarm init --advertise-addr 10.60.8.99
```

Sur Noeud2 et Noeud3 :
```bash
docker swarm join --token <TOKEN> 10.60.8.99:2377
```

Promotion en managers (haute disponibilité) :
```bash
docker node promote Noeud2 Noeud3
docker node ls
```

---

## 4. Stockage NFS

### Serveur (Noeud3)
```bash
apt install -y nfs-kernel-server
mkdir -p /srv/nfs/swarm/{registry,mariadb,web,vscode}
chown -R nobody:nogroup /srv/nfs/swarm
chmod -R 777 /srv/nfs/swarm

echo "/srv/nfs/swarm 10.60.8.0/24(rw,sync,no_subtree_check,no_root_squash)" > /etc/exports
exportfs -ra
systemctl enable --now nfs-kernel-server
exportfs -v
```

### Clients (Noeud1, Noeud2)
```bash
apt install -y nfs-common
showmount -e 10.60.8.101
```

---

## 5. Secrets Docker

Les mots de passe ne sont pas stockés dans `stack.yml` mais dans des Docker secrets :
```bash
printf '<MOT_DE_PASSE>' | docker secret create db_root_password -
printf '<MOT_DE_PASSE>' | docker secret create db_password -
printf '<MOT_DE_PASSE>' | docker secret create vscode_password -
docker secret ls
```

Les services les lisent via `/run/secrets/<nom>` :
- MariaDB : `MARIADB_ROOT_PASSWORD_FILE`, `MARIADB_PASSWORD_FILE`
- VSCode : `FILE__PASSWORD`
- PHP : `file_get_contents('/run/secrets/db_password')`

---

## 6. Déploiement du stack

### Fichier `stack.yml`
```yaml
version: "3.8"

services:

  registry:
    image: registry:2
    ports:
      - "5000:5000"
    volumes:
      - registry_data:/var/lib/registry
    networks:
      - backend
    deploy:
      replicas: 1
      restart_policy:
        condition: any

  mariadb:
    image: mariadb:11
    environment:
      MARIADB_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
      MARIADB_DATABASE: appdb
      MARIADB_USER: appuser
      MARIADB_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_root_password
      - db_password
    volumes:
      - mariadb_data:/var/lib/mysql
    networks:
      - backend
    deploy:
      replicas: 1
      restart_policy:
        condition: any

  php:
    image: php:8.3-fpm
    command: sh -c "docker-php-ext-install mysqli pdo_mysql && php-fpm"
    secrets:
      - db_password
    volumes:
      - web_data:/var/www/html
    networks:
      - backend
    deploy:
      replicas: 2
      restart_policy:
        condition: any

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - web_data:/var/www/html
    networks:
      - backend
    command:
      - sh
      - -c
      - |
        cat > /etc/nginx/conf.d/default.conf <<'EOF'
        server {
          listen 80;
          root /var/www/html;
          index index.php index.html;
          location / { try_files $$uri $$uri/ /index.php?$$query_string; }
          location ~ \.php$$ {
            fastcgi_pass php:9000;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $$document_root$$fastcgi_script_name;
          }
        }
        EOF
        exec nginx -g 'daemon off;'
    deploy:
      replicas: 2
      restart_policy:
        condition: any

  vscode:
    image: lscr.io/linuxserver/code-server:latest
    environment:
      PUID: "1000"
      PGID: "1000"
      TZ: Europe/Paris
      FILE__PASSWORD: /run/secrets/vscode_password
      DEFAULT_WORKSPACE: /config/workspace
    secrets:
      - vscode_password
    ports:
      - "8443:8443"
    volumes:
      - vscode_data:/config
      - web_data:/config/workspace
    networks:
      - backend
    deploy:
      replicas: 1
      restart_policy:
        condition: any

secrets:
  db_root_password:
    external: true
  db_password:
    external: true
  vscode_password:
    external: true

networks:
  backend:
    driver: overlay
    attachable: true

volumes:
  registry_data:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=10.60.8.101,rw,nfsvers=4"
      device: ":/srv/nfs/swarm/registry"
  mariadb_data:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=10.60.8.101,rw,nfsvers=4"
      device: ":/srv/nfs/swarm/mariadb"
  web_data:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=10.60.8.101,rw,nfsvers=4"
      device: ":/srv/nfs/swarm/web"
  vscode_data:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=10.60.8.101,rw,nfsvers=4"
      device: ":/srv/nfs/swarm/vscode"
```

Points clés :
- `restart_policy: any` : Swarm relance automatiquement tout conteneur arrêté.
- PHP et Nginx en **2 réplicas** : répartition de charge et tolérance à la perte d'un conteneur.
- La configuration Nginx est générée au démarrage (`$$` = `$` échappé pour Compose), ce qui permet de tout garder dans un seul fichier.
- Les volumes sont montés en **NFS** depuis Noeud3 : un conteneur recréé sur n'importe quel nœud retrouve ses données.

### Déploiement
```bash
cd ~/swarm
docker stack deploy -c stack.yml swarm
docker service ls
docker stack ps swarm
```

Accès :
- Web : `http://10.60.8.99`
- VSCode : `http://10.60.8.99:8443`
- Registry : `http://localhost:5000/v2/_catalog`

Grâce au routing mesh, les services sont accessibles depuis l'IP de n'importe quel nœud.

---

## 7. Sauvegarde

Script `scripts/backup-swarm.sh` (installé dans `/usr/local/bin/` sur Noeud3) :
- dump de toutes les bases MariaDB via un conteneur temporaire sur le réseau `swarm_backend` (fonctionne quel que soit le nœud hébergeant MariaDB) ;
- archive des volumes NFS `registry`, `web`, `vscode` (sockets exclus) ;
- rétention de 7 jours.

Le mot de passe root est lu depuis `/root/.db_root_password` (chmod 600).

Planification (cron, chaque nuit à 2h) :
```
0 2 * * * /usr/local/bin/backup-swarm.sh >> /var/log/backup-swarm.log 2>&1
```

---

## 8. Procédures de test

### 8.1 Fonctionnement des services

| Test | Commande | Résultat attendu | Statut |
|------|----------|------------------|--------|
| État du cluster | `docker node ls` | 3 nœuds `Ready`, 1 `Leader`, 2 `Reachable` | ✅ |
| État des services | `docker service ls` | Tous les réplicas à `1/1` ou `2/2` | ✅ |
| Registry | `docker push localhost:5000/alpine` puis `curl http://localhost:5000/v2/_catalog` | `{"repositories":["alpine"]}` | ✅ |
| Nginx + PHP + MariaDB | `curl http://10.60.8.99` | `Servi par : <id>` + `DB OK` | ✅ |
| Répartition de charge | Recharger la page plusieurs fois | L'ID du conteneur PHP change | ✅ |
| VSCode | `http://10.60.8.99:8443` | Interface accessible, `index.php` visible | ✅ |
| Persistance NFS | `ls /srv/nfs/swarm/*` sur Noeud3 | Données présentes dans chaque dossier | ✅ |

### 8.2 Simulation de la perte d'un conteneur

```bash
docker service ps swarm_mariadb -f desired-state=running   # repérer le nœud
docker kill $(docker ps -q -f name=swarm_mariadb)          # sur ce nœud
watch docker service ps swarm_mariadb
```

**Résultat** : l'ancienne tâche passe en `Failed`, Swarm recrée automatiquement le conteneur en quelques secondes. La page web affiche toujours `DB OK` : les données sont conservées sur le NFS. ✅

### 8.3 PCA – Perte d'un nœud manager

```bash
poweroff                 # sur Noeud1 (Leader)
docker node ls           # sur Noeud2, après ~20 s
docker service ls
```

**Résultat** :
- Noeud1 passe en `Down / Unreachable`.
- Noeud3 est élu nouveau `Leader`, le cluster reste administrable.
- Les services de Noeud1 sont redéployés sur Noeud2 et Noeud3.
- Au redémarrage, Noeud1 rejoint le cluster en `Reachable` et ses anciennes tâches sont arrêtées. ✅

> Les compteurs temporairement supérieurs (ex. `2/1`) pendant la panne sont normaux : Swarm ne peut pas confirmer l'arrêt des tâches d'un nœud injoignable.

### 8.4 PRA – Restauration de la base

Simulation de la perte de données :
```bash
docker run --rm --network swarm_backend mariadb:11 \
  mariadb -h mariadb -uroot -p<MOT_DE_PASSE> -e "DROP DATABASE appdb;"
```
→ la page web affiche `Unknown database 'appdb'`.

Restauration :
```bash
docker run --rm -i --network swarm_backend mariadb:11 \
  mariadb -h mariadb -uroot -p<MOT_DE_PASSE> < /backup/<DATE>/mariadb.sql
```
→ la page web affiche de nouveau `DB OK`. ✅

---

## 9. Limites connues

- **NFS = point unique de défaillance** : si Noeud3 tombe, les volumes deviennent inaccessibles. Pistes : NFS répliqué (DRBD + keepalived), GlusterFS ou Ceph.
- **MariaDB en une seule instance** : la disponibilité repose sur le redémarrage automatique par Swarm. Piste : cluster Galera.
- **Sauvegardes stockées sur Noeud3** : à externaliser (autre machine, stockage distant).
- **Registry en HTTP** : à sécuriser avec TLS et authentification.

---

## 10. Arborescence du dépôt

```
swarm/
├── README.md
├── stack.yml
├── scripts/
│   └── backup-swarm.sh
└── web/
    └── index.php
```
