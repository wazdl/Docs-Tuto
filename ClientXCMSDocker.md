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

## ⚙️ Étape 4 : Configuration du fichier `.env`

Éditez le fichier de configuration principal avec l'éditeur de texte `nano` :

```bash
sudo nano .env
```

Remplissez les informations dans le fichier en respectant strictement le format suivant :

### 1. Paramètres de l'application & OAuth
Remplacez les valeurs d'exemple par vos informations réelles :

```ini
# Renseignez votre URL complète : https://votre-domaine.com (Production) OU http://VOTRE_IP_LOCALE (Local)
APP_URL=https://votre-domaine.com

# Renseignez les clés obtenues à l'Étape 1
OAUTH_CLIENT_ID=votre_client_id_ici
OAUTH_CLIENT_SECRET=votre_secret_id_ici

# Mettez "production" si vous avez un nom de domaine, ou "local" si vous utilisez une IP
APP_ENV=production
```

### 2. Configuration de la Base de Données (⚠️ Ne pas modifier)
Pour fonctionner correctement avec le réseau interne du fichier `docker-compose.yml`, ces valeurs doivent rester intactes :

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

---

## 🚀 Étape 5 : Lancement et Déploiement

1. Démarrez les conteneurs Docker en arrière-plan. Docker va lire le fichier `.env` et configurer l'environnement :
   ```bash
   sudo docker compose up -d
   ```

2. Patientez environ **30 secondes** le temps que la base de données s'initialise correctement, puis exécutez ces commandes de configuration initiale :

   * **Générer la clé de sécurité de l'application :**
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
     *(Suivez attentivement les instructions affichées sur votre terminal pour créer vos identifiants d'accès au panel d'administration).*

---

## 🌐 Étape 6 : Connexion au Panel

1. Ouvrez votre navigateur internet.
2. Rendez-vous sur l'adresse configurée dans votre `APP_URL` suivie de `/admin/login` (Exemple : `https://votre-domaine.com/admin/login`).
3. Connectez-vous avec les identifiants configurés à l'étape précédente. Grâce aux clés OAuth renseignées dans le fichier `.env`, votre licence sera validée automatiquement avec le serveur de licence officiel !

🎉 **Félicitations !** Votre installation est maintenant terminée, sécurisée et survivra aux redémarrages de votre serveur !

---

## 🛠️ Guide de Dépannage pour les développeurs (IP Locale)

> [!WARNING]
> Si vous installez ClientXCMS sans nom de domaine (via une adresse IP directe en `http://`), votre navigateur bloquera la transmission sécurisée des cookies de session. Cela provoquera une **Erreur 419 (Page Expired)** à la connexion.

Pour corriger ce comportement spécifique aux environnements de test locaux :

1. Assurez-vous d'avoir bien configuré ces lignes dans votre fichier `.env` :
   ```ini
   APP_ENV=local
   SESSION_DRIVER=cookie
   ```

2. Lancez cette commande pour forcer l'application à accepter les connexions HTTP non-sécurisées *(à réexécuter uniquement si vous recréez ou réinitialisez le conteneur)* :
   ```bash
   sudo docker compose exec -i app sh -c "cat > .env" < .env && sudo docker compose exec app php artisan optimize:clear
   ```
