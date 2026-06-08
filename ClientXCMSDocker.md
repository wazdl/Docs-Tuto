# 🐳 Guide d'Installation Officiel : ClientXCMS via Docker

Ce guide vous accompagne pas-à-pas pour déployer **ClientXCMS** à l'aide de Docker. Conçu pour être totalement autonome et persistant après redémarrage, il couvre les installations en production (avec nom de domaine et SSL) ainsi qu'en développement (IP locale).

---

## 📋 Prérequis

* Un serveur Linux (Ubuntu/Debian hautement recommandé) propre.
* Les ports `80` (HTTP) et `443` (HTTPS) ouverts sur votre pare-feu.
* 🔑 **Vos clés OAuth ClientXCMS** (Disponibles dans votre espace client officiel).

---

## 🔑 Étape 1 : Récupération de vos clés OAuth

Avant de commencer, vous devez lier votre future installation à votre compte ClientXCMS :

1. Connectez-vous à votre [espace client sur le site officiel de ClientXCMS](https://clientxcms.com).
2. Allez dans la section de gestion de votre produit / licence.
3. Récupérez vos deux clés :
   * **OAuth Client ID**
   * **OAuth Secret**
   *(Ce sont de longues chaînes de caractères complexes. Gardez-les précieusement de côté).*

---

## 🛠️ Étape 2 : Préparation du Système

1. Mettez à jour le serveur et installez les outils de base indispensables :
   ```bash
   sudo apt update && sudo apt upgrade -y
   sudo apt install git nano curl -y
   ```

2. Installez Docker et Docker Compose (version officielle) :
   ```bash
   sudo curl -fsSL https://get.docker.com -o get-docker.sh
   sudo sh get-docker.sh
   ```

---

## 📂 Étape 3 : Récupération du Projet

Créez votre dossier de travail et préparez les fichiers de configuration nécessaires :

```bash
# Création du répertoire web standard
sudo mkdir -p /var/www
cd /var/www

# Clonage du dépôt officiel de ClientXCMS
sudo git clone https://github.com/ClientXCMS/clientxcms.git
cd clientxcms

# Création des fichiers de configuration à partir des exemples fournis
sudo cp docker-compose.example.yml docker-compose.yml
sudo cp .env.example .env
```

---

## ⚙️ Étape 4 : Configuration

> [!WARNING]
> **Point critique — `APP_URL` doit être définie à un seul endroit.**
> L'`APP_URL` ne doit être configurée **que dans le fichier `.env`**. Si elle est également présente dans le `docker-compose.yml` avec une valeur par défaut (ex: `${APP_URL:-http://localhost}`), Docker fera une **concaténation des deux valeurs**, ce qui provoquera des erreurs de licence et d'authentification OAuth. La ligne dans le `docker-compose.yml` doit être `APP_URL: "${APP_URL}"` sans valeur par défaut.

### Fichier `.env`

Éditez le fichier de configuration principal :

```bash
sudo nano .env
```

Remplissez les informations en respectant le format suivant :

#### 1. Paramètres de l'application & OAuth

```ini
# Renseignez votre URL complète : https://votre-domaine.com (Production) OU http://VOTRE_IP_LOCALE (Local)
APP_URL=http://192.168.1.100

# Renseignez les clés obtenues à l'Étape 1
OAUTH_CLIENT_ID=votre_client_id_ici
OAUTH_CLIENT_SECRET=votre_secret_id_ici

# Mettez "production" si vous avez un nom de domaine, ou "local" si vous utilisez une IP
APP_ENV=production
```

#### 2. Configuration de la Base de Données (⚠️ Ne pas modifier)

```ini
DB_CONNECTION=mysql
DB_HOST=database
DB_PORT=3306
DB_DATABASE=clientxcms
DB_USERNAME=clientxcms
DB_PASSWORD=clientxcms
```

> [!TIP]
> Pour sauvegarder et quitter sur `nano` : faites `Ctrl + X`, puis confirmez avec `Y` (ou `O` en français), et enfin appuyez sur `Entrée`.

### Fichier `docker-compose.yml`

Deux points importants à vérifier :

Enlever `APP_URL: "http://localhost"` pour éviter la concaténation et le disfonctionnement des modules.

**1. `APP_URL` sans valeur par défaut :**
```yaml
environment:
    APP_KEY: "${APP_KEY}"
    APP_URL: "${APP_URL}"   # ← Pas de valeur par défaut après le :-
    APP_ENV: "${APP_ENV:-production}"
    # ... reste de la configuration
```

**2. Entrypoint pour les permissions des modules et chemin correct du volume :**
```yaml
services:
  app:
    image: clientxcms/panel:master
    restart: always
    entrypoint: ["/bin/sh", "-c", "chmod -R 777 /app/modules /app/storage /app/resources/themes && chown -R www-data:www-data /app/modules /app/storage /app/resources/themes && exec /bin/ash .github/docker/entrypoint.sh supervisord -n -c /etc/supervisord.conf"]
    volumes:
      - .github/logs/nginx/:/var/log/nginx/
      - ./modules:/app/modules        # ← /app/modules et NON /var/www/html/modules
      - ./storage:/app/storage
      - ./resources/themes:/app/resources/themes
      - bootstrap-cache:/app/bootstrap/cache # Persistance pour garder les modules activés / désactivés
```
```
A la fin du fichier :
    volumes:
      mariadb-data:
      bootstrap-cache:
```

> [!NOTE]
> L'entrypoint règle automatiquement les permissions du dossier `modules` à chaque démarrage du conteneur, sans intervention manuelle.

---

## 🚀 Étape 5 : Lancement et Déploiement

1. Démarrez les conteneurs Docker en arrière-plan :
   ```bash
   sudo docker compose up -d
   ```

2. Patientez environ **30 secondes** le temps que la base de données s'initialise, puis exécutez ces commandes :

   * **Afficher la clé de sécurité** *(copiez-la pour la mettre dans votre `.env` à la variable `APP_KEY`)* :
     ```bash
     sudo docker exec -it clientxcms-app-1 php artisan key:generate --show
     ```

   * **Générer et enregistrer automatiquement la clé de sécurité :**
     ```bash
     sudo docker compose exec app php artisan key:generate
     ```

   * **Vider et optimiser les caches de Laravel :**
     ```bash
     sudo docker compose exec app php artisan optimize:clear
     ```

   * **Créer le compte Administrateur :**
     ```bash
     sudo docker compose exec app php artisan clientxcms:install-admin
     ```
     *(Suivez les instructions affichées pour créer vos identifiants d'accès).*

---

## 🌐 Étape 6 : Connexion au Panel

1. Ouvrez votre navigateur et rendez-vous sur l'adresse configurée dans `APP_URL` suivie de `/admin/login`.
   Exemple : `http://192.168.1.100/admin/login`
2. Connectez-vous avec les identifiants créés à l'étape précédente.

🎉 **Félicitations !** Votre installation est terminée et survivra aux redémarrages !

---

## 🧩 Installation d'une Extension / Module personnalisé

> [!NOTE]
> Les fichiers de l'application sont situés dans **/app** à l'intérieur du conteneur (et non `/var/www/html`). Le volume du `docker-compose.yml` doit donc pointer vers `/app/modules` (déjà configuré à l'Étape 4).

### 1. Placer votre module

Copiez votre dossier de module dans `./modules/` sur la machine hôte. Vérifiez qu'il est bien visible dans le conteneur :

```bash
sudo docker compose exec app ls /app/modules/
```

### 2. Enregistrer le module en base de données

ClientXCMS stocke la liste des modules actifs en base. Un module présent dans le dossier mais absent de la table `modules` ne sera pas chargé. Enregistrez-le manuellement :

```bash
sudo docker compose exec app php artisan tinker --execute="
DB::table('modules')->insert([
    'name'       => 'NomDuModule',
    'uuid'       => 'uuid-du-module',
    'version'    => '1.0',
    'enabled'    => 1,
    'provider'   => 'App\\\\Modules\\\\NomDuModule\\\\NomDuModuleServiceProvider',
    'created_at' => now(),
    'updated_at' => now(),
]);
echo 'Module enregistré avec succès';
"
```

> Remplacez `NomDuModule` et `uuid-du-module` par les valeurs de votre module (consultez son fichier `module.json`).

Pour vérifier la structure exacte de la table si besoin :
```bash
sudo docker compose exec app php artisan tinker --execute="var_dump(DB::getSchemaBuilder()->getColumnListing('modules'));"
```

### 3. Régénérer l'autoload et vider les caches

```bash
sudo docker compose exec app composer dump-autoload
sudo docker compose exec app php artisan optimize:clear
```

Votre module devrait maintenant apparaître dans l'interface d'administration.

---

## 🛠️ Guide de Dépannage

### Erreur 419 (Page Expired) en environnement local

Si vous installez ClientXCMS sans nom de domaine (via une adresse IP en `http://`), votre navigateur peut bloquer les cookies de session.

1. Dans `docker-compose.yml`, ajoutez sous `SESSION_DRIVER` :
   ```yaml
   SESSION_SECURE_COOKIE: "${SESSION_SECURE_COOKIE:-false}"
   SESSION_SECURE: "${SESSION_SECURE:-false}"
   ```

2. Forcez l'injection du `.env` dans le conteneur (si vous faites des modifications) :
   ```bash
   sudo docker compose cp .env app:/var/www/html/.env && sudo docker compose exec app php artisan optimize:clear
   ```

### Erreur "Token has been revoked"

Cette erreur indique que vos clés OAuth sont invalides ou absentes. Vérifiez dans votre `.env` :
```ini
OAUTH_CLIENT_ID=votre_vrai_client_id
OAUTH_CLIENT_SECRET=votre_vrai_secret
```
Puis rechargez la configuration :
```bash
sudo docker compose exec app php artisan config:clear
sudo docker compose exec app php artisan cache:clear
```

---

## 💡 Commandes Utiles

### Redémarrer les conteneurs (après un changement de `.env`)
```bash
sudo docker compose down
sudo docker compose up -d
```

### Vider et optimiser le cache
```bash
sudo docker compose exec app php artisan optimize:clear
```

### Injecter le fichier `.env` local dans le conteneur
```bash
sudo docker compose cp .env app:/var/www/html/.env
```

### Vérifier les modules enregistrés en base
```bash
sudo docker compose exec app php artisan tinker --execute="var_dump(DB::table('modules')->get()->toArray());"
```
