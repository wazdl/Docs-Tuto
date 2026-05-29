# 🚀 Guide d'Installation : ClientXCMS via Docker

> Installation fluide et fonctionnelle de ClientXCMS sur un serveur vierge (Ubuntu/Debian).  
> Inclut les correctifs essentiels pour les environnements de développement (IP locale) afin d'éviter l'erreur **419 Page Expired**.

---

## 📋 Prérequis

- Un serveur Linux (Ubuntu/Debian recommandé) fraîchement installé
- Accès `root` ou `sudo`

---

## Étape 1 — Préparation du Serveur

Mettez à jour le système et installez les outils de base :

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install git nano curl -y
```

Installez Docker et Docker Compose via le script officiel :

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

---

## Étape 2 — Téléchargement des Fichiers

Créez le répertoire de travail et clonez le dépôt officiel :

```bash
sudo mkdir -p /var/www
cd /var/www
sudo git clone https://github.com/ClientXCMS/clientxcms.git
cd clientxcms
```

Copiez les fichiers de configuration d'exemple :

```bash
sudo cp docker-compose.example.yml docker-compose.yml
sudo cp .env.example .env
```

---

## Étape 3 — Configuration du fichier `.env` ⚠️

> C'est ici que se jouent la majorité des erreurs d'installation. Lisez attentivement.

```bash
sudo nano .env
```

### 1. URL de l'application (`APP_URL`)

| Environnement | Valeur |
|---|---|
| Domaine avec HTTPS | `APP_URL=https://votre-domaine.com` |
| IP locale (HTTP) | `APP_URL=http://VOTRE_ADRESSE_IP` |

### 2. Correctif Anti-Erreur 419

Ajoutez ou modifiez ces lignes pour que les sessions fonctionnent correctement, surtout sans HTTPS actif :

```env
SESSION_DRIVER=file
SESSION_SECURE_COOKIE=false
SESSION_SECURE=false
```

### 3. Base de données

Définissez un mot de passe robuste :

```env
DB_PASSWORD=votre_mot_de_passe_robuste
```

> 💾 **Sauvegarder avec Nano :** `Ctrl+X` → `Y` → `Entrée`

---

## Étape 4 — Déploiement et Génération de la Clé

Lancez l'environnement Docker :

```bash
sudo docker compose --env-file .env up -d --build
```

> ⏳ Patientez environ une minute pour laisser la base de données s'initialiser.

Générez la clé de sécurité obligatoire (chiffrement des sessions) :

```bash
sudo docker exec -it clientxcms-app-1 php artisan key:generate
```

> **Note :** Si la clé ne s'ajoute pas automatiquement dans le `.env`, exécutez :
> ```bash
> sudo docker exec -it clientxcms-app-1 php artisan key:generate --show
> ```
> Copiez la valeur `base64:...` et collez-la manuellement à la ligne `APP_KEY=` de votre `.env`.

Videz le cache pour appliquer la clé :

```bash
sudo docker exec -it clientxcms-app-1 php artisan optimize:clear
```

---

## Étape 5 — Compte Administrateur et Licence

Créez votre compte administrateur :

```bash
sudo docker exec -it clientxcms-app-1 php artisan clientxcms:install-admin
```

Suivez les instructions à l'écran (email, mot de passe, etc.).

**Accès au site :** Ouvrez votre navigateur (de préférence en navigation privée pour le premier test) et rendez-vous sur l'adresse configurée dans `APP_URL`.

**Activation :** L'écran de validation de licence s'affiche. Cliquez sur le bouton d'activation pour relier votre site à votre compte ClientXCMS.

> Si vous utilisez une IP locale, assurez-vous de l'autoriser dans votre espace client sur le site officiel.

---

## Étape 6 — Automatisation des Mises à Jour du `.env`

Docker et Laravel mettent les configurations en cache — une simple modification du `.env` ne sera **pas** prise en compte sans redémarrage. Ce script automatise la procédure.

### Création du script

```bash
nano maj_env.sh
```

Collez le contenu suivant :

```bash
#!/bin/bash
echo "🔄 Application des modifications du fichier .env..."
sudo docker compose down
sudo docker compose --env-file .env up -d --force-recreate
echo "🧹 Nettoyage du cache de l'application..."
sudo docker exec -it clientxcms-app-1 php artisan optimize:clear
echo "✅ Mise à jour terminée avec succès !"
```

Rendez le script exécutable :

```bash
chmod +x maj_env.sh
```

### Utilisation

À chaque modification du `.env` (clé API Stripe, changement de domaine, etc.), placez-vous dans `/var/www/clientxcms` et exécutez :

```bash
./maj_env.sh
```

Le système redémarre et applique automatiquement vos nouveaux paramètres.

---

## 🗂️ Récapitulatif des commandes utiles

| Action | Commande |
|---|---|
| Démarrer les conteneurs | `sudo docker compose --env-file .env up -d` |
| Arrêter les conteneurs | `sudo docker compose down` |
| Voir les logs | `sudo docker compose logs -f` |
| Vider le cache | `sudo docker exec -it clientxcms-app-1 php artisan optimize:clear` |
| Appliquer un nouveau `.env` | `./maj_env.sh` |

---

## 🐛 Problèmes fréquents

**Erreur 419 Page Expired**  
→ Vérifiez que `SESSION_SECURE_COOKIE=false` et `SESSION_SECURE=false` sont bien définis dans le `.env`, puis relancez `./maj_env.sh`.

**`APP_KEY` manquante**  
→ Lancez `php artisan key:generate --show` et copiez manuellement la valeur dans le `.env`.

**La base de données ne démarre pas**  
→ Attendez 60 secondes après le `docker compose up` avant de lancer les commandes `artisan`.
