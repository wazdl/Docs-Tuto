# 🦤 Pelican Panel — Guide d'installation complet sur Debian (Docker)

> Guide complet pour déployer **Pelican Panel** avec Docker, Portainer et MariaDB sur une machine Debian.
> Conçu pour les débutants — chaque commande est expliquée.

---

## 📖 Table des matières

1. [Comprendre Docker et les concepts clés](#-comprendre-docker-et-les-concepts-clés)
2. [Prérequis](#-prérequis)
3. [Étape 1 — Installation de Docker](#étape-1--installation-de-docker)
4. [Étape 2 — Installation de Portainer](#étape-2--installation-de-portainer)
5. [Étape 3 — Organisation des services](#étape-3--organisation-des-services)
6. [Étape 4 — Déploiement de Pelican Panel](#étape-4--déploiement-de-pelican-panel)
7. [Étape 5 — Récupération de la clé d'application](#étape-5--récupération-de-la-clé-dapplication-)
8. [Étape 6 — Finalisation via l'interface Web](#étape-6--finalisation-via-linterface-web)
9. [Gérer plusieurs services avec Portainer](#-gérer-plusieurs-services-avec-portainer)
10. [Commandes utiles](#️-commandes-utiles)
11. [Dépannage](#-dépannage)

---

## 🧠 Comprendre Docker et les concepts clés

Avant de commencer, voici les notions essentielles à connaître :

| Terme | Explication simple |
|---|---|
| **Docker** | Un outil qui permet de faire tourner des applications dans des "boîtes" isolées appelées conteneurs |
| **Conteneur** | Une boîte autonome qui contient une application et tout ce dont elle a besoin pour fonctionner |
| **Image** | Le modèle à partir duquel un conteneur est créé (comme un moule) |
| **Docker Compose** | Un outil pour décrire et lancer plusieurs conteneurs ensemble via un fichier `compose.yml` |
| **Stack** | Un ensemble de conteneurs définis dans un même fichier `compose.yml` et déployés ensemble |
| **Volume** | Un espace de stockage persistant pour les données d'un conteneur (les données survivent aux redémarrages) |
| **Réseau** | Un réseau virtuel interne qui permet aux conteneurs de communiquer entre eux |
| **Portainer** | Une interface web pour gérer Docker visuellement, sans avoir à taper des commandes |

> 💡 **En résumé :** Docker permet de faire tourner des services (comme Pelican, une base de données, etc.) de façon isolée et propre sur votre machine, sans risquer de casser le système.

---

## 📋 Prérequis

- Une machine sous **Debian** (VM ou serveur physique)
- Accès **SSH** avec droits `sudo` ou connexion en `root`
- L'**adresse IP** de votre machine (notée `X.X.X.X` dans ce guide)
  - Pour la trouver : `ip a | grep inet`
- Une connexion internet active sur le serveur

---

## Étape 1 — Installation de Docker

Ces commandes installent Docker et son plugin Compose sur votre système Debian.

```bash
# 1. Mise à jour de la liste des paquets disponibles
sudo apt update && sudo apt upgrade -y

# 2. Installation des outils nécessaires pour ajouter le dépôt Docker
sudo apt install -y ca-certificates curl gnupg lsb-release

# 3. Ajout de la clé GPG officielle de Docker (pour vérifier l'authenticité des paquets)
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 4. Ajout du dépôt officiel Docker dans les sources APT
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 5. Installation de Docker et du plugin Compose
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**Vérification :** Après l'installation, testez que Docker fonctionne correctement :

```bash
sudo docker run hello-world
```

> ✅ Si vous voyez un message `Hello from Docker!`, l'installation est réussie.

**Optionnel — Utiliser Docker sans `sudo` :**

```bash
# Ajoute votre utilisateur au groupe docker (évite de taper sudo à chaque fois)
sudo usermod -aG docker $USER

# Déconnectez-vous puis reconnectez-vous en SSH pour que le changement prenne effet
```

---

## Étape 2 — Installation de Portainer

**Portainer** est une interface web qui vous permet de gérer tous vos conteneurs Docker visuellement : démarrer, arrêter, consulter les logs, déployer de nouvelles stacks... sans jamais taper une commande.

```bash
# 1. Création d'un volume dédié pour les données de Portainer
sudo docker volume create portainer_data

# 2. Lancement du conteneur Portainer
sudo docker run -d \
  --name portainer \
  --restart=always \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

> 💡 **Explication des options :**
> - `-p 9000:9000` → rend l'interface accessible sur le port 9000
> - `-v /var/run/docker.sock:/var/run/docker.sock` → donne à Portainer l'accès à Docker
> - `--restart=always` → Portainer redémarre automatiquement avec le serveur

**Accès à Portainer :**

Ouvrez votre navigateur et rendez-vous sur :
```
http://<IP_DE_VOTRE_VM>:9000
```

Lors du premier accès, créez un compte administrateur. Choisissez un mot de passe robuste.

> ⚠️ **Important :** Portainer vous demande de créer votre compte dans les premières minutes. Passé ce délai, il se verrouille par sécurité. Si c'est trop tard, redémarrez le conteneur : `sudo docker restart portainer`

---

## Étape 3 — Organisation des services

Quand on gère **plusieurs services Docker**, il est essentiel de bien organiser ses fichiers pour ne pas s'y perdre. Voici la structure recommandée :

```
/opt/
├── portainer/          ← Portainer (déjà déployé via docker run)
├── pelican/            ← Pelican Panel + sa base de données
│   └── compose.yml
├── mon-autre-service/  ← Un autre service futur
│   └── compose.yml
└── encore-un-autre/    ← Et ainsi de suite...
    └── compose.yml
```

> 💡 **Règle d'or :** Un dossier = un service (ou un groupe de services liés). Cela permet de démarrer, arrêter ou mettre à jour chaque service indépendamment.

Créez le dossier pour Pelican :

```bash
mkdir -p /opt/pelican && cd /opt/pelican
```

---

## Étape 4 — Déploiement de Pelican Panel

Créez le fichier de configuration Docker Compose :

```bash
nano compose.yml
```

Copiez-collez le contenu suivant. **Remplacez chaque occurrence de `X.X.X.X` par l'adresse IP réelle de votre machine**, ainsi que les mots de passe par des valeurs personnalisées :

```yaml
services:

  # ─────────────────────────────────────────────
  # Conteneur 1 : Pelican Panel (l'application web)
  # ─────────────────────────────────────────────
  panel:
    image: ghcr.io/pelican-dev/panel:latest
    restart: always
    networks:
      - default
    ports:
      - "80:80"       # HTTP
      - "443:443"     # HTTPS
    extra_hosts:
      - "host.docker.internal:host-gateway"
    volumes:
      - pelican-data:/pelican-data              # Données de config persistantes
      - pelican-logs:/var/www/html/storage/logs # Logs persistants
    environment:
      XDG_DATA_HOME: /pelican-data
      APP_URL: "http://X.X.X.X"           # ← IP de votre serveur
      ADMIN_EMAIL: "votre-email@example.com"
      DB_CONNECTION: mysql
      DB_HOST: "X.X.X.X"                  # ← IP de votre VM
      DB_PORT: 3306
      DB_DATABASE: pelican
      DB_USERNAME: pelican
      DB_PASSWORD: motDePasseDB            # ← À personnaliser
    depends_on:
      - mysql   # Le panel attend que MySQL soit prêt avant de démarrer

  # ─────────────────────────────────────────────
  # Conteneur 2 : MariaDB (la base de données)
  # ─────────────────────────────────────────────
  mysql:
    image: mariadb:10.11
    restart: always
    networks:
      - default
    ports:
      - "3306:3306"   # Expose MySQL à l'hôte pour la configuration initiale
    environment:
      MYSQL_ROOT_PASSWORD: motDePasseDBROOT!   # ← À personnaliser (accès root)
      MYSQL_DATABASE: pelican                   # Nom de la base créée automatiquement
      MYSQL_USER: pelican                       # Utilisateur dédié à Pelican
      MYSQL_PASSWORD: motDePasseMYSQL           # ← À personnaliser
    volumes:
      - pelican-db:/var/lib/mysql   # Les données de la BDD survivent aux redémarrages

# Déclaration des volumes (espaces de stockage persistants)
volumes:
  pelican-data:
  pelican-logs:
  pelican-db:

# Déclaration du réseau interne pour que les conteneurs communiquent
networks:
  default:
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

> 💾 Sauvegardez avec `Ctrl+O` → `Entrée` → `Ctrl+X`

Lancez les conteneurs :

```bash
sudo docker compose up -d
```

> ⏳ **Au premier démarrage**, Docker télécharge les images (peut prendre quelques minutes selon votre connexion) et initialise la base de données. Attendez **1 à 2 minutes** avant d'accéder à l'interface web.

Vérifiez que les conteneurs tournent bien :

```bash
sudo docker compose ps
```

Vous devriez voir `panel` et `mysql` avec le statut `running`.

---

## Étape 5 — Récupération de la clé d'application ⚠️

> **Cette étape est critique.** La clé d'application chiffre vos données sensibles. Sans elle, vous ne pourrez pas restaurer le panel en cas de problème.

```bash
sudo docker compose exec panel cat /pelican-data/.env | grep APP_KEY
```

Vous obtiendrez une ligne de ce type :
```
APP_KEY=base64:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx=
```

**Conservez cette clé précieusement** dans un gestionnaire de mots de passe (Bitwarden, KeePass...) ou un fichier chiffré.

---

## Étape 6 — Finalisation via l'interface Web

1. Ouvrez votre navigateur et accédez à l'assistant d'installation :

   ```
   http://<IP_DE_VOTRE_VM>/installer
   ```

   > 💡 Utilisez un onglet en **navigation privée** pour éviter les conflits de cache.

2. Suivez les étapes de l'assistant à l'écran.

3. À l'étape **configuration de la base de données**, renseignez :

   | Champ | Valeur à saisir |
   |---|---|
   | Pilote | `MySQL` ou `MariaDB` |
   | Hôte | L'IP de votre VM (ex: `192.168.0.10`) |
   | Nom de la base | `pelican` |
   | Port | `3306` |
   | Nom d'utilisateur | `pelican` |
   | Mot de passe | La valeur de `DB_PASSWORD` dans votre `compose.yml` |

4. Cliquez sur **Suivant** pour terminer l'assistant et créer votre compte administrateur.

---

## 🖥️ Gérer plusieurs services avec Portainer

Portainer vous permet de gérer toutes vos stacks Docker depuis une interface web claire. Voici comment l'utiliser au quotidien.

### Déployer une nouvelle stack depuis Portainer

1. Connectez-vous à Portainer : `http://<IP_DE_VOTRE_VM>:9000`
2. Dans le menu de gauche, cliquez sur **Stacks**
3. Cliquez sur **+ Add stack**
4. Donnez un nom à votre stack (ex: `pelican`)
5. Dans l'éditeur, collez le contenu de votre `compose.yml`
6. Cliquez sur **Deploy the stack**

### Vue d'ensemble des stacks

Depuis **Stacks**, vous voyez en un coup d'œil tous vos services :

```
Stacks
├── pelican        ✅ running  (2 conteneurs : panel, mysql)
├── jellyfin       ✅ running  (1 conteneur)
├── nextcloud      ⏹ stopped  (3 conteneurs)
└── monitoring     ✅ running  (2 conteneurs)
```

### Actions disponibles par stack

| Action | Comment faire dans Portainer |
|---|---|
| Voir les conteneurs | Cliquer sur le nom de la stack |
| Démarrer / Arrêter | Boutons en haut à droite de la stack |
| Consulter les logs | Conteneurs → icône 📋 à droite du conteneur |
| Mettre à jour une image | Conteneurs → Recreate → cocher "Pull latest image" |
| Modifier le compose.yml | Stacks → cliquer sur la stack → Editor |
| Supprimer une stack | Stacks → cocher → Remove |

### Bonnes pratiques avec plusieurs services

> 💡 **Une stack = un dossier** — Gardez toujours une copie de vos `compose.yml` sur votre machine locale, pas uniquement dans Portainer.

> 💡 **Les ports doivent être uniques** — Deux services ne peuvent pas utiliser le même port. Exemple : si Pelican utilise le port `80`, un autre service devra utiliser `8080`, `8081`, etc.

> 💡 **Nommez vos volumes explicitement** — Préfixez les noms de volumes avec le nom du service (ex: `pelican-db`, `jellyfin-config`) pour éviter les confusions.

> 💡 **Redémarrage automatique** — Toujours ajouter `restart: always` à vos services critiques pour qu'ils redémarrent automatiquement après un reboot du serveur.

---

## 🛠️ Commandes utiles

Ces commandes sont à exécuter **depuis le dossier du service** (ex: `cd /opt/pelican`).

### Gestion de la stack

| Action | Commande |
|---|---|
| Démarrer tous les conteneurs | `sudo docker compose up -d` |
| Arrêter tous les conteneurs | `sudo docker compose down` |
| Redémarrer tous les conteneurs | `sudo docker compose restart` |
| Voir l'état des conteneurs | `sudo docker compose ps` |
| Consulter les logs en temps réel | `sudo docker compose logs -f` |
| Logs d'un seul service | `sudo docker compose logs -f panel` |

### Mises à jour

```bash
# Télécharger les nouvelles versions des images
sudo docker compose pull

# Recréer les conteneurs avec les nouvelles images
sudo docker compose up -d

# Supprimer les anciennes images inutilisées (libère de l'espace)
sudo docker image prune -f
```

### Maintenance de Pelican

```bash
# Vider le cache de l'application
sudo docker compose exec panel php artisan config:clear

# Exécuter les migrations de base de données (après une mise à jour)
sudo docker compose exec panel php artisan migrate --force
```

### Gestion globale de Docker

```bash
# Voir tous les conteneurs actifs sur le serveur
sudo docker ps

# Voir tous les conteneurs (y compris arrêtés)
sudo docker ps -a

# Voir l'espace utilisé par Docker
sudo docker system df

# Nettoyage général (conteneurs arrêtés, images inutiles, réseaux orphelins)
sudo docker system prune -f
```

---

## 🔧 Dépannage

### Les conteneurs ne démarrent pas

```bash
# Consultez les logs pour identifier l'erreur
sudo docker compose logs
```

### Impossible d'accéder à l'interface web

- Vérifiez que le conteneur `panel` est bien en état `running` : `sudo docker compose ps`
- Vérifiez que l'IP dans `APP_URL` et `DB_HOST` correspond bien à votre machine
- Vérifiez qu'aucun pare-feu ne bloque le port 80 : `sudo ufw status`

### Erreur de connexion à la base de données

- Attendez 30 secondes supplémentaires après le lancement (MariaDB peut être lent à initialiser)
- Vérifiez que les mots de passe dans `compose.yml` sont identiques entre `panel` et `mysql`
- Vérifiez que `DB_HOST` contient bien l'IP de votre VM et non `localhost`

### Portainer ne répond plus

```bash
sudo docker restart portainer
```

### Un port est déjà utilisé

```bash
# Trouver quel processus utilise un port (ex: port 80)
sudo ss -tlnp | grep :80
```
