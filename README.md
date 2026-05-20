# 🦤 Pelican Panel — Guide d'installation sur Debian (Docker)

> Guide d'installation complet pour déployer **Pelican Panel** avec Docker et MariaDB sur une machine Debian.

---

## 📋 Prérequis

- Une machine Debian (VM ou serveur physique)
- Accès SSH avec droits `sudo` ou `root`
- L'adresse IP de votre VM (notée `X.X.X.X` dans ce guide)

---

## Étape 1 — Préparation du système et installation de Docker

Connectez-vous en SSH à la machine Debian et exécutez les commandes suivantes pour installer **Docker CE** et son plugin Compose :

```bash
# Mise à jour des paquets du système
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg lsb-release

# Ajout de la clé GPG officielle de Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Ajout du dépôt Docker dans les sources APT
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Installation de Docker et du plugin Compose
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

---

## Étape 2 — Création de la structure et du fichier de configuration

Créez un répertoire dédié à l'application :

```bash
mkdir -p /opt/pelican && cd /opt/pelican
```

Créez le fichier de déploiement Docker Compose :

```bash
nano compose.yml
```

Remplissez le fichier avec le contenu ci-dessous. **Remplacez impérativement `X.X.X.X` par l'IP réelle de votre VM.**

```yaml
services:
  panel:
    image: ghcr.io/pelican-dev/panel:latest
    restart: always
    networks:
      - default
    ports:
      - "80:80"
      - "443:443"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    volumes:
      - pelican-data:/pelican-data
      - pelican-logs:/var/www/html/storage/logs
    environment:
      XDG_DATA_HOME: /pelican-data
      APP_URL: "http://X.X.X.X"         # ← Remplacer par l'IP de votre serveur
      ADMIN_EMAIL: "votre-email@example.com"
      DB_CONNECTION: mysql
      DB_HOST: "X.X.X.X"               # ← Remplacer par l'IP de votre VM
      DB_PORT: 3306
      DB_DATABASE: pelican
      DB_USERNAME: pelican
      DB_PASSWORD: motDePasseDB
    depends_on:
      - mysql

  mysql:
    image: mariadb:10.11
    restart: always
    networks:
      - default
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: motDePasseDBROOT!
      MYSQL_DATABASE: pelican
      MYSQL_USER: pelican
      MYSQL_PASSWORD: motDePasseMYSQL
    volumes:
      - pelican-db:/var/lib/mysql

volumes:
  pelican-data:
  pelican-logs:
  pelican-db:

networks:
  default:
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

> 💾 Sauvegardez et quittez avec `Ctrl+O` → `Entrée` → `Ctrl+X`

---

## Étape 3 — Lancement de l'environnement

Démarrez les conteneurs en arrière-plan :

```bash
sudo docker compose up -d
```

> ⏳ **Note :** Au tout premier démarrage, Docker télécharge les images et initialise la base de données. Attendez **1 à 2 minutes** avant d'accéder à l'interface web.

---

## Étape 4 — Récupération de la clé d'application ⚠️

> **Important :** Conservez précieusement cette clé — elle est indispensable pour restaurer le panel en cas de problème.

```bash
sudo docker compose exec panel cat /pelican-data/.env | grep APP_KEY
```

Copiez la ligne obtenue (de la forme `APP_KEY=CLE...`) dans un endroit sûr (gestionnaire de mots de passe, fichier chiffré, etc.).

---

## Étape 5 — Finalisation via l'interface Web

1. Ouvrez votre navigateur et rendez-vous sur :

   ```
   http://<IP_DE_VOTRE_VM>/installer
   ```

   > 💡 Utilisez un onglet en **navigation privée** pour éviter les conflits de cache HTTP/HTTPS.

2. Suivez les étapes de l'assistant à l'écran.

3. À l'étape de configuration de la base de données, renseignez :

   | Champ | Valeur |
   |---|---|
   | Pilote | MySQL / MariaDB |
   | Hôte | `<IP_DE_VOTRE_VM>` (ex: `192.168.0.10`) |
   | Nom de la base | `pelican` |
   | Port | `3306` |
   | Nom d'utilisateur | `pelican` |
   | Mot de passe | `motDePasseDB` |

4. Cliquez sur **Suivant** pour terminer l'assistant et créer votre compte administrateur.

---

## 🛠️ Commandes utiles

| Action | Commande |
|---|---|
| Arrêter le panel | `sudo docker compose down` |
| Redémarrer le panel | `sudo docker compose restart` |
| Consulter les logs en temps réel | `sudo docker compose logs -f` |
| Vider le cache de l'application | `sudo docker compose exec panel php artisan config:clear` |
