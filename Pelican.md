# 🦤 Pelican Panel — Guide d'installation complet sur Debian (Docker)

> Guide d'installation pas à pas pour déployer **Pelican Panel + Wings** avec Docker et MariaDB sur une machine Debian.  
> Conçu pour les débutants — chaque commande est expliquée.

---

## 📚 Table des matières

1. [Comprendre Docker et Docker Compose](#-comprendre-docker-et-docker-compose)
2. [Prérequis](#-prérequis)
3. [Étape 1 — Installation de Docker](#étape-1--installation-de-docker)
4. [Étape 2 — Organiser ses services Docker](#étape-2--organiser-ses-services-docker)
5. [Étape 3 — Configurer Pelican Panel + Wings](#étape-3--configurer-pelican-panel--wings)
6. [Étape 4 — Lancer l'environnement](#étape-4--lancer-lenvironnement)
7. [Étape 5 — Sauvegarder la clé d'application ⚠️](#étape-5--sauvegarder-la-clé-dapplication-)
8. [Étape 6 — Finalisation via l'interface Web](#étape-6--finalisation-via-linterface-web)
9. [Étape 7 — Configurer Wings depuis le Panel](#étape-7--configurer-wings-depuis-le-panel)
10. [Gérer plusieurs services Docker Compose](#-gérer-plusieurs-services-docker-compose)
11. [Commandes utiles](#️-commandes-utiles)
12. [Résolution des problèmes courants](#-résolution-des-problèmes-courants)

---

## 🧠 Comprendre Docker et Docker Compose

Avant de commencer, voici quelques notions clés :

| Terme | Explication simple |
|---|---|
| **Docker** | Un outil qui permet de faire tourner des applications dans des "boîtes" isolées appelées **conteneurs** |
| **Conteneur** | Une boîte autonome qui contient une application et tout ce dont elle a besoin pour fonctionner |
| **Image** | Le "modèle" à partir duquel un conteneur est créé (ex: l'image `mariadb` contient MariaDB prêt à l'emploi) |
| **Docker Compose** | Un outil pour décrire et lancer plusieurs conteneurs ensemble via un simple fichier texte (`compose.yml`) |
| **Volume** | Un espace de stockage persistant : même si le conteneur est supprimé, les données restent |
| **Stack** | Un ensemble de conteneurs définis dans un même fichier `compose.yml` |
| **Port** | Une "porte d'entrée" réseau. `80:80` signifie : le port 80 de la machine → port 80 du conteneur |

> 💡 **En résumé :** Docker Compose lit votre fichier `compose.yml`, télécharge les images nécessaires, et lance tous vos conteneurs automatiquement.

### Pelican Panel vs Wings — quelle différence ?

| Composant | Rôle |
|---|---|
| **Panel** | L'interface web d'administration — vous gérez vos serveurs de jeu depuis ici |
| **Wings** | Le daemon qui tourne sur le nœud et exécute réellement les serveurs de jeu dans des conteneurs Docker |

> 💡 En résumé : le Panel donne les ordres, Wings les exécute. Les deux doivent pouvoir communiquer entre eux.

---

## ✅ Prérequis

- Une machine sous **Debian** (VM ou serveur physique)
- Accès **SSH** avec droits `sudo` ou `root`
- L'**adresse IP** de votre VM (notée `X.X.X.X` dans ce guide)

> 💡 Pour trouver votre IP : `ip a` ou `hostname -I`

---

## Étape 1 — Installation de Docker

Connectez-vous en SSH à votre machine Debian, puis exécutez les commandes suivantes :

### 1.1 — Mise à jour du système

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg lsb-release
```

> Ces commandes mettent à jour la liste des paquets disponibles et installent des outils de base nécessaires pour la suite.

### 1.2 — Ajout du dépôt officiel Docker

```bash
# Crée le dossier pour stocker la clé de sécurité de Docker
sudo install -m 0755 -d /etc/apt/keyrings

# Télécharge et enregistre la clé GPG officielle de Docker
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Ajoute le dépôt Docker à la liste des sources APT
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

> Docker n'est pas dans les dépôts Debian par défaut. Ces commandes ajoutent le dépôt officiel de Docker pour pouvoir l'installer.

### 1.3 — Installation de Docker

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

> - `docker-ce` : le moteur Docker
> - `docker-ce-cli` : les commandes en ligne (ex: `docker ps`)
> - `docker-compose-plugin` : permet d'utiliser `docker compose` (avec espace, la version moderne)

### 1.4 — Vérifier l'installation

```bash
docker --version
docker compose version
```

Vous devriez obtenir quelque chose comme :
```
Docker version 26.x.x, build ...
Docker Compose version v2.x.x
```

---

## Étape 2 — Organiser ses services Docker

### Pourquoi organiser ses fichiers ?

Quand on gère plusieurs services (Pelican, Nextcloud, Vaultwarden, etc.), il est important d'avoir une structure claire. Sans organisation, il devient vite difficile de savoir quel fichier correspond à quel service.

### Structure recommandée

```
/opt/
├── pelican/          ← Dossier du service Pelican (Panel + Wings)
│   └── compose.yml
├── nextcloud/        ← Exemple d'un autre service
│   └── compose.yml
└── vaultwarden/      ← Exemple d'un autre service
    └── compose.yml
```

> 💡 **Règle d'or :** 1 dossier = 1 service = 1 fichier `compose.yml`  
> Cela vous permet de démarrer, arrêter ou modifier un service **sans toucher aux autres**.

### Créer le dossier pour Pelican

```bash
mkdir -p /opt/pelican
cd /opt/pelican
```

> `mkdir -p` crée le dossier et tous les dossiers parents si nécessaire. `cd` se déplace dans ce dossier.

---

## Étape 3 — Configurer Pelican Panel + Wings

Créez le fichier de configuration Docker Compose :

```bash
nano compose.yml
```

> `nano` est un éditeur de texte simple dans le terminal. Si vous préférez, vous pouvez utiliser `vim` ou tout autre éditeur.

Copiez-collez le contenu suivant, puis **remplacez `X.X.X.X` par l'IP réelle de votre VM** (à tous les endroits indiqués) :

```yaml
services:
  panel:
    image: ghcr.io/pelican-dev/panel:latest
    restart: always
    networks:
      - default
    ports:
      - "80:80"     # Port HTTP — accès web au panel
      - "443:443"   # Port HTTPS
    extra_hosts:
      - "host.docker.internal:host-gateway"
    volumes:
      - pelican-data:/pelican-data               # Données de config du panel
      - pelican-logs:/var/www/html/storage/logs  # Logs de l'application
    environment:
      XDG_DATA_HOME: /pelican-data
      APP_URL: "http://X.X.X.X"         # ← Remplacer par l'IP de votre serveur
      ADMIN_EMAIL: "votre-email@example.com"
      DB_CONNECTION: mysql
      DB_HOST: "X.X.X.X"               # ← Remplacer par l'IP de votre VM
      DB_PORT: 3306
      DB_DATABASE: pelican
      DB_USERNAME: pelican
      DB_PASSWORD: motDePasseDB         # ← Choisissez un mot de passe fort
    depends_on:
      - mysql   # Le panel ne démarre qu'une fois MySQL prêt

  mysql:
    image: mariadb:10.11
    restart: always
    networks:
      - default
    ports:
      - "3306:3306"   # Port MySQL exposé sur la machine hôte
    environment:
      MYSQL_ROOT_PASSWORD: motDePasseDBROOT!   # ← Mot de passe root (à changer !)
      MYSQL_DATABASE: pelican
      MYSQL_USER: pelican
      MYSQL_PASSWORD: motDePasseMYSQL          # ← Doit correspondre à DB_PASSWORD ci-dessus
    volumes:
      - pelican-db:/var/lib/mysql   # Les données de la BDD survivent aux redémarrages

  wings:
    container_name: wings
    image: ghcr.io/pelican-dev/wings:latest
    restart: always
    networks:
      - default
    ports:
      - "8080:8080"   # Port de communication Panel ↔ Wings
      - "2022:2022"   # Port SFTP pour les fichiers des serveurs de jeu
    extra_hosts:
      - "host.docker.internal:host-gateway"
      - "X.X.X.X:host-gateway"   # ← Remplacer par l'IP de votre VM (communication interne)
    tty: true
    environment:
      TZ: "Europe/Paris"     # Fuseau horaire
      WINGS_UID: 988         # UID utilisateur sous lequel Wings s'exécute
      WINGS_GID: 988         # GID groupe sous lequel Wings s'exécute
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock   # Accès au daemon Docker de l'hôte
      - /var/lib/pelican:/var/lib/pelican           # Fichiers des serveurs de jeu
      - /etc/pelican:/etc/pelican                   # Config Wings (config.yml généré par le panel)
    # ⚠️ Wings attend sagement que /etc/pelican/config.yml soit créé par le panel
    # avant de démarrer correctement — c'est normal au premier lancement

volumes:
  pelican-data:    # Données persistantes du panel
  pelican-logs:    # Logs persistants
  pelican-db:      # Base de données persistante
  # Note : les données Wings sont stockées directement sur l'hôte
  # (/var/lib/pelican et /etc/pelican) plutôt que dans des volumes Docker nommés

networks:
  default:
    ipam:
      config:
        - subnet: 172.20.0.0/16   # Réseau interne isolé pour cette stack
```

> 💾 Sauvegardez et quittez : `Ctrl+O` → `Entrée` → `Ctrl+X`

### Comprendre les paramètres spécifiques à Wings

- **`/var/run/docker.sock`** : Wings a besoin d'accéder au daemon Docker de la machine hôte pour lancer les conteneurs des serveurs de jeu. C'est ce montage qui lui donne ce pouvoir.
- **`/var/lib/pelican`** : c'est ici que sont stockés les fichiers de tous vos serveurs de jeu (maps, configs, sauvegardes...).
- **`/etc/pelican`** : Wings y lira son fichier `config.yml`, qui sera généré automatiquement depuis l'interface web du panel à l'étape 7.
- **`extra_hosts` avec votre IP** : permet à Wings de résoudre l'adresse du panel sur le réseau interne Docker — indispensable pour qu'ils communiquent.
- **`port 8080`** : le panel utilise ce port pour envoyer ses ordres à Wings (créer/démarrer/arrêter des serveurs).
- **`port 2022`** : port SFTP permettant de transférer des fichiers vers vos serveurs de jeu.

---

## Étape 4 — Lancer l'environnement

Depuis le dossier `/opt/pelican`, lancez :

```bash
sudo docker compose up -d
```

> - `up` : crée et démarre les conteneurs
> - `-d` : mode "détaché" — les conteneurs tournent en arrière-plan (vous gardez la main dans le terminal)

Docker va télécharger les images puis démarrer les conteneurs. Vous verrez quelque chose comme :

```
✔ Container pelican-mysql-1    Started
✔ Container pelican-panel-1    Started
✔ Container wings              Started
```

> ⏳ **Au premier démarrage**, Docker télécharge les images et initialise la base de données. Attendez **1 à 2 minutes** avant d'accéder à l'interface web.
>
> ⚠️ **Wings affichera des erreurs au premier démarrage** — c'est tout à fait normal ! Il attend que le fichier `config.yml` soit généré par le panel (étape 7). Il ne crashe pas, il patiente.

Vérifiez que tout tourne correctement :

```bash
sudo docker compose ps
```

Les trois conteneurs doivent afficher le statut `running`.

---

## Étape 5 — Sauvegarder la clé d'application ⚠️

Cette étape est **indispensable**. La clé d'application chiffre les données sensibles. Sans elle, vous ne pourrez pas restaurer votre panel en cas de problème.

```bash
sudo docker compose exec panel cat /pelican-data/.env | grep APP_KEY
```

Vous obtiendrez une ligne de ce type :

```
APP_KEY=xXxXxXxXxXxXxXxXxXxXxXxXxXxXxXxXxXxXxXxXxX=
```

> 🔒 **Copiez cette ligne** dans un endroit sûr : gestionnaire de mots de passe (Bitwarden, KeePass...), fichier chiffré, etc.

---

## Étape 6 — Finalisation via l'interface Web

1. Ouvrez votre navigateur et rendez-vous sur :

   ```
   http://<IP_DE_VOTRE_VM>/installer
   ```

   > 💡 Utilisez un onglet en **navigation privée** pour éviter les conflits de cache HTTP/HTTPS.

2. Suivez les étapes de l'assistant.

3. À l'étape **base de données**, renseignez :

   | Champ | Valeur |
   |---|---|
   | Pilote | MySQL / MariaDB |
   | Hôte | `<IP_DE_VOTRE_VM>` (ex: `192.168.50.132`) |
   | Nom de la base | `pelican` |
   | Port | `3306` |
   | Nom d'utilisateur | `pelican` |
   | Mot de passe | `motDePasseDB` |

4. Cliquez sur **Suivant** pour terminer l'assistant et créer votre compte administrateur.

---

## Étape 7 — Configurer Wings depuis le Panel

Une fois connecté à l'interface d'administration, il faut créer un **nœud** (node) et générer la configuration de Wings.

### 7.1 — Créer un nœud

1. Dans le menu d'administration, allez dans **Nodes** → **Create New**.
2. Remplissez les informations :

   | Champ | Valeur |
   |---|---|
   | Name | Un nom pour votre nœud (ex: `node-local`) |
   | FQDN | L'IP de votre VM (ex: `192.168.50.132`) |
   | Communicate Over SSL | Non (HTTP pour un setup local) |
   | Daemon Port | `8080` |
   | Daemon SFTP Port | `2022` |

3. Sauvegardez le nœud.

### 7.2 — Générer et déployer la configuration Wings

1. Sur la page du nœud que vous venez de créer, cliquez sur l'onglet **Configuration**.
2. Vous verrez un bloc YAML — c'est la configuration que Wings attend. Copiez-le.
3. Sur votre machine, créez le fichier de configuration :

   ```bash
   sudo mkdir -p /etc/pelican
   sudo nano /etc/pelican/config.yml
   ```

4. Collez le contenu YAML copié depuis le panel, sauvegardez (`Ctrl+O` → `Entrée` → `Ctrl+X`).

### 7.3 — Redémarrer Wings

Maintenant que le fichier de config existe, redémarrez Wings pour qu'il le prenne en compte :

```bash
cd /opt/pelican
sudo docker compose restart wings
```

Vérifiez les logs pour confirmer que Wings démarre correctement :

```bash
sudo docker compose logs -f wings
```

Vous devriez voir quelque chose comme :
```
[INFO] wings is running and ready to accept connections
```

### 7.4 — Vérifier la connexion Panel ↔ Wings

De retour dans l'interface web, sur la page du nœud, l'indicateur de statut doit passer au **vert**. Si c'est le cas, le panel et Wings communiquent correctement.

---

## 📦 Gérer plusieurs services Docker Compose

### Principe fondamental

Chaque service vit dans son propre dossier. Pour interagir avec un service, il faut **se placer dans son dossier** avant d'exécuter les commandes Docker Compose.

```bash
# Exemple : gérer Pelican
cd /opt/pelican
sudo docker compose up -d       # Démarrer
sudo docker compose down        # Arrêter
sudo docker compose logs -f     # Voir les logs

# Exemple : gérer Nextcloud (indépendamment de Pelican)
cd /opt/nextcloud
sudo docker compose up -d
sudo docker compose down
```

> ✅ Stopper Pelican n'affecte pas Nextcloud, et inversement. Chaque stack est totalement indépendante.

---

### Voir tous les conteneurs en cours d'exécution

Pour avoir une vue globale de **tous** vos conteneurs (toutes stacks confondues) :

```bash
sudo docker ps
```

Exemple de sortie :

```
CONTAINER ID   IMAGE                          STATUS          NAMES
a1b2c3d4e5f6   ghcr.io/pelican-dev/panel      Up 2 hours      pelican-panel-1
b2c3d4e5f6a7   mariadb:10.11                  Up 2 hours      pelican-mysql-1
c3d4e5f6a7b8   ghcr.io/pelican-dev/wings      Up 2 hours      wings
```

---

### Éviter les conflits de ports entre services

Chaque service qui expose un port doit utiliser un port **unique** sur la machine hôte. Vous ne pouvez pas avoir deux services qui écoutent sur le même port en même temps.

**Exemple de répartition :**

| Service | Port machine | Port conteneur | URL d'accès |
|---|---|---|---|
| Pelican Panel | `80` | `80` | `http://IP` |
| Wings (API) | `8080` | `8080` | Utilisé en interne par le panel |
| Wings (SFTP) | `2022` | `2022` | Accès SFTP aux fichiers |
| Nextcloud | `8081` | `80` | `http://IP:8081` |
| Portainer | `9000` | `9000` | `http://IP:9000` |

> 💡 Si vous avez un **reverse proxy** (comme Nginx Proxy Manager ou Traefik), vous pouvez utiliser des sous-domaines à la place des ports. C'est la solution recommandée pour un setup propre à long terme.

---

### Éviter les conflits de réseaux internes

Chaque stack doit avoir son propre **sous-réseau** pour éviter les conflits :

```yaml
# Stack Pelican
networks:
  default:
    ipam:
      config:
        - subnet: 172.20.0.0/16

# Stack Nextcloud
networks:
  default:
    ipam:
      config:
        - subnet: 172.21.0.0/16

# Stack Vaultwarden
networks:
  default:
    ipam:
      config:
        - subnet: 172.22.0.0/16
```

> Chaque stack communique en interne sur son propre réseau isolé. Les stacks ne se voient pas entre elles, ce qui est plus sécurisé.

---

### Mettre à jour un service

Pour mettre à jour l'image d'un service vers la dernière version :

```bash
cd /opt/pelican

# Télécharger la nouvelle version de l'image
sudo docker compose pull

# Recréer les conteneurs avec la nouvelle image
sudo docker compose up -d

# Supprimer les anciennes images devenues inutiles
sudo docker image prune -f
```

> ✅ Vos données sont dans des **volumes** et ne sont pas affectées par la mise à jour.

---

### Supprimer complètement un service

Si vous voulez supprimer un service et **garder les données** :

```bash
cd /opt/pelican
sudo docker compose down
```

Si vous voulez supprimer un service et **effacer toutes les données** (attention, irréversible !) :

```bash
sudo docker compose down -v
```

> ⚠️ L'option `-v` supprime les volumes Docker nommés (base de données, config panel...).  
> Les données Wings stockées dans `/var/lib/pelican` et `/etc/pelican` **ne sont pas supprimées** par cette commande — elles sont sur l'hôte. Supprimez-les manuellement si nécessaire : `sudo rm -rf /var/lib/pelican /etc/pelican`

---

## 🛠️ Commandes utiles

### Commandes Docker Compose (à exécuter depuis le dossier du service)

| Action | Commande |
|---|---|
| Démarrer les conteneurs | `sudo docker compose up -d` |
| Arrêter les conteneurs | `sudo docker compose down` |
| Redémarrer les conteneurs | `sudo docker compose restart` |
| Redémarrer seulement Wings | `sudo docker compose restart wings` |
| Voir l'état des conteneurs | `sudo docker compose ps` |
| Voir les logs en temps réel | `sudo docker compose logs -f` |
| Voir les logs de Wings | `sudo docker compose logs -f wings` |
| Voir les logs du panel | `sudo docker compose logs -f panel` |
| Mettre à jour les images | `sudo docker compose pull && sudo docker compose up -d` |
| Supprimer avec les données | `sudo docker compose down -v` |

### Commandes Docker globales (tous les services)

| Action | Commande |
|---|---|
| Voir tous les conteneurs actifs | `sudo docker ps` |
| Voir tous les conteneurs (y compris arrêtés) | `sudo docker ps -a` |
| Voir les images téléchargées | `sudo docker images` |
| Voir l'espace disque utilisé | `sudo docker system df` |
| Nettoyer les images inutilisées | `sudo docker image prune -f` |
| Nettoyage complet (attention !) | `sudo docker system prune -f` |

### Commandes spécifiques à Pelican

| Action | Commande |
|---|---|
| Vider le cache de l'application | `sudo docker compose exec panel php artisan config:clear` |
| Accéder au shell du conteneur panel | `sudo docker compose exec panel bash` |
| Accéder au shell du conteneur Wings | `sudo docker compose exec wings bash` |
| Récupérer la clé d'application | `sudo docker compose exec panel cat /pelican-data/.env \| grep APP_KEY` |
| Vérifier la config Wings | `cat /etc/pelican/config.yml` |

---

## 🔧 Résolution des problèmes courants

### Wings ne démarre pas / affiche des erreurs au premier lancement

C'est normal. Wings a besoin que `/etc/pelican/config.yml` existe avant de démarrer correctement. Suivez l'étape 7 pour générer ce fichier depuis le panel, puis redémarrez Wings :

```bash
sudo docker compose restart wings
sudo docker compose logs -f wings
```

### Le panel ne voit pas Wings (statut rouge dans l'interface)

Vérifiez les points suivants :
1. Le port `8080` est bien exposé et Wings tourne : `sudo docker compose ps`
2. L'IP dans `extra_hosts` de Wings correspond à l'IP réelle de votre VM
3. Le champ **FQDN** du nœud dans le panel pointe vers la bonne IP
4. Consultez les logs Wings : `sudo docker compose logs wings`

### Le panel ne démarre pas

```bash
# Regardez les logs pour identifier l'erreur
sudo docker compose logs panel
```

### Erreur "port is already allocated"

Un autre service utilise déjà ce port sur votre machine.

```bash
# Identifier quel processus utilise le port 80
sudo ss -tlnp | grep :80
# Identifier quel processus utilise le port 8080
sudo ss -tlnp | grep :8080
```

Changez le port dans votre `compose.yml` puis relancez.

### Erreur de connexion à la base de données

Vérifiez que :
1. L'IP dans `DB_HOST` correspond bien à l'IP de votre VM
2. Le mot de passe dans `DB_PASSWORD` correspond à `MYSQL_PASSWORD`
3. Le conteneur MySQL est bien démarré : `sudo docker compose ps`

### Réinitialiser complètement et repartir de zéro

```bash
cd /opt/pelican

# Arrêter et supprimer les conteneurs ET les volumes Docker (données panel/BDD perdues)
sudo docker compose down -v

# Supprimer aussi les données Wings sur l'hôte (optionnel)
sudo rm -rf /var/lib/pelican /etc/pelican

# Relancer proprement
sudo docker compose up -d
```

---

*Guide rédigé pour Pelican Panel — [Documentation officielle](https://pelican.dev)*
