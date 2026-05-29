# 📦 ClientXCMS — Guide d'Installation Docker

Méthode complète pas-à-pas pour déployer **ClientXCMS** avec Docker, avec les correctifs intégrés pour les erreurs courantes (1045, 419).

**Environnement & Stack :** Ubuntu / Debian | Docker Compose | MariaDB | Laravel | HTTP & HTTPS

---

## 00. Prérequis

Avant de commencer, vérifiez que votre environnement remplit ces conditions :
* 🖥️ Un serveur Linux propre (Ubuntu 22.04 ou Debian 11+ recommandé)
* 🔑 Accès `sudo` ou `root` sur la machine
* 🌐 Les ports `80` et `443` libres et accessibles
* 🔌 Connexion Internet active pour télécharger les images Docker

> [!NOTE]
> Ce guide intègre les correctifs communautaires pour les erreurs **1045 Access Denied** (MariaDB) et **419 Page Expired** (session Laravel en HTTP). Suivez les étapes dans l'ordre pour éviter ces problèmes.

---

## 01. Préparation du Système

Mettez à jour les paquets du système et installez les outils requis :
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install git nano curl -y
```

Installez ensuite Docker et Docker Compose via le script officiel :
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

> [!TIP]
> Vérifiez l'installation avec `docker --version` et `docker compose version`. Les deux commandes doivent retourner un numéro de version.

---

## 02. Récupération du Projet

Créez le dossier de travail, clonez le dépôt et préparez les fichiers de configuration :
```bash
# Création du dossier et clonage
sudo mkdir -p /var/www
cd /var/www
sudo git clone https://github.com/ClientXCMS/clientxcms.git
cd clientxcms

# Création des fichiers à partir des exemples
cp docker-compose.example.yml docker-compose.yml
cp .env.example .env
```

---

## 03. Configuration du fichier `.env`

Ouvrez le fichier `.env` avec l'éditeur Nano :
```bash
sudo nano .env
```

Configurez les trois sections suivantes. Respectez scrupuleusement les valeurs indiquées :

### 1 — URL de l'application

| Cas d'usage | Valeur à saisir |
| :--- | :--- |
| **IP locale / test HTTP** | `APP_URL=http://192.168.X.X` |
| **Production / HTTPS** | `APP_URL=https://votre-domaine.com` |

### 2 — Base de données

> [!WARNING]
> Le `docker-compose.yml` communautaire fixe les identifiants de la base de données sur `clientxcms`. Ne modifiez aucune de ces valeurs, sinon vous obtiendrez une erreur **1045 Access Denied**.

```env
DB_CONNECTION=mysql
DB_HOST=database
DB_PORT=3306
DB_DATABASE=clientxcms
DB_USERNAME=clientxcms
DB_PASSWORD=clientxcms
```

### 3 — Correctif Anti-Erreur 419 (HTTP uniquement)

Si vous testez sans HTTPS, ces paramètres débloquent les formulaires de connexion :
```env
SESSION_DRIVER=cookie
SESSION_SECURE_COOKIE=false
SESSION_SECURE=false
```

> [!TIP]
> Pour sauvegarder et quitter Nano : `Ctrl+X`, puis `Y`, puis `Entrée`.

---

## 04. Lancement et Patch de Session

Démarrez les conteneurs en arrière-plan :
```bash
sudo docker compose up -d
```

> [!NOTE]
> Pour vérifier l'état des conteneurs : `sudo docker compose ps`. Tous les services doivent afficher le statut **running** (et non *restarting*).

---

## 05. Initialisation de ClientXCMS

La clé de chiffrement (`APP_KEY`) générée par Laravel dans le conteneur ne se sauvegarde pas automatiquement dans votre fichier `.env` sur le serveur hôte. Voici comment la générer, la voir et l'appliquer correctement :

### 1. Générer et afficher la clé :
```bash
sudo docker compose exec app php artisan key:generate --show
```
*(Copiez précieusement la clé qui s'affiche à l'écran, elle commence généralement par `base64:`)*

### 2. Ajouter la clé à votre fichier `.env` :
```bash
sudo nano .env
```
Ajoutez cette ligne dans le fichier avec la clé que vous venez de copier :
```env
APP_KEY=base64:votre_cle_copiee_ici
```
Sauvegardez (`Ctrl+X`, `Y`, `Entrée`).

### 3. Appliquer la clé et finaliser l'installation :
Maintenant que la clé est dans votre fichier hôte, utilisez votre script de mise à jour pour l'injecter et vider les caches, puis créez le compte admin :
```bash
# Injecte le nouveau .env et vide les caches
./maj_env.sh

# Crée le compte administrateur (suivez les instructions)
sudo docker compose exec app php artisan clientxcms:install-admin
```

> [!IMPORTANT]
> La dernière commande vous demandera de saisir un nom d'utilisateur, une adresse email et un mot de passe. Conservez ces informations précieusement.

---

## 06. Premier Accès et Activation de la Licence

1. **Ouvrir un onglet de navigation privée**  
   Indispensable pour éviter les anciens cookies persistants qui pourraient bloquer la connexion.
2. **Accéder à l'interface d'administration**  
   Rendez-vous sur l'adresse configurée dans votre `APP_URL`, par ex. : `http://192.168.X.X/admin/login`
3. **Se connecter avec vos identifiants**  
   Utilisez l'email et le mot de passe créés lors de la commande `install-admin`.
4. **Activer la licence**  
   Sur l'écran de licence, cliquez sur le bouton vert. Si vous êtes sur une IP locale, autorisez temporairement cette IP dans votre espace client sur le site officiel de ClientXCMS.

---

## 🔄 Script de Mise à Jour du `.env`

Par défaut, Docker et Laravel n'appliquent pas les modifications du `.env` à chaud. Ce script automatise la resynchronisation complète.

Créez le fichier `maj_env.sh` :
```bash
nano maj_env.sh
```

Collez ce contenu :
```bash
#!/bin/bash
echo "🔄 Application des modifications du .env..."
sudo docker compose down
sudo docker compose --env-file .env up -d --force-recreate
echo "🧹 Injection et nettoyage du cache..."
sudo docker compose exec -i app sh -c "cat > .env" < .env
sudo docker compose exec app php artisan optimize:clear
echo "✅ Configuration synchronisée avec succès !"
```

Rendez le script exécutable :
```bash
sudo chmod +x maj_env.sh
```

> [!TIP]
> Désormais, après chaque modification du `.env`, lancez simplement `./maj_env.sh` depuis le dossier du projet.

---

## 🛠️ Guide de Dépannage

### Erreur 1045 — Access Denied for user 'clientxcms'
* **Cause :** Vous avez modifié `DB_PASSWORD` ou `DB_DATABASE`. Le conteneur MariaDB n'accepte que les valeurs par défaut `clientxcms`.
* **Solution :** Remettez `clientxcms` partout dans la section Database de votre `.env`, puis purgez le volume défectueux :
  ```bash
  sudo docker compose down -v
  ./maj_env.sh
  ```
  *(Attention : Le flag `-v` supprime la base de données. N'utilisez-le qu'en cas d'installation fraîche).*

### Erreur 419 — Page Expired
* **Cause :** Formulaire de connexion bloqué. Votre navigateur refuse le cookie de session car l'application réclame du HTTPS alors que vous êtes en HTTP.
* **Solution :**
  1. Vérifiez que `SESSION_SECURE=false` et `SESSION_SECURE_COOKIE=false` sont bien dans votre `.env`.
  2. Relancez `./maj_env.sh`.
  3. Utilisez impérativement un onglet de navigation privée.

### Page blanche / Erreur 500
* **Cause :** Problème de cache de configuration après l'initialisation.
* **Solution :** Videz les caches :
  ```bash
  sudo docker compose exec app php artisan optimize:clear
  sudo docker compose exec app php artisan config:clear
  sudo docker compose exec app php artisan cache:clear
  ```

### Permission denied sur les fichiers
* **Cause :** Erreur d'écriture dans `storage/` ou `bootstrap/cache/`.
* **Solution :** Corrigez les permissions :
  ```bash
  sudo docker compose exec app chmod -R 775 storage bootstrap/cache
  sudo docker compose exec app chown -R www-data:www-data storage bootstrap/cache
  ```

---

## ✅ Checklist de Vérification

Utilisez cette checklist pour vous assurer que tout est en ordre avant d'aller en production :

- [x] Docker et Docker Compose installés et fonctionnels
- [x] Ports 80 et 443 ouverts sur le firewall
- [x] APP_URL correspond exactement à votre IP ou domaine
- [x] DB_* : toutes les valeurs sont à "clientxcms"
- [x] SESSION_SECURE=false (si vous testez en HTTP)
- [x] Conteneurs en statut "running" (pas "restarting")
- [x] Clé applicative générée
- [x] Cache vidé
- [x] Compte admin créé
- [x] Licence activée dans l'interface admin
- [ ] **(Production)** Certificat SSL Let's Encrypt configuré
- [ ] **(Production)** SESSION_SECURE=true activé
- [ ] **(Production)** Sauvegardes automatiques configurées

---

🎉 **Installation terminée**

Votre instance ClientXCMS est opérationnelle. Pour toute question, rejoignez la communauté sur le Discord officiel ou consultez la documentation officielle.

*Pensez à désactiver les options HTTP (`SESSION_SECURE=false`) lorsque vous passez en production avec un certificat SSL valide.*
