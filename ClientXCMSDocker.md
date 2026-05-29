# 🚀 ClientXCMS — Installation Professionnelle via Docker

> Guide d'installation sécurisé, pas-à-pas, sur un serveur vierge (Ubuntu).
> Inclut le correctif **erreur 419 Page Expired** et l'isolation de la base de données.

---

## 🔒 Sécurité appliquée

| Mesure | Détail |
|---|---|
| **Mots de passe dynamiques** | L'application et la base de données se synchronisent automatiquement via le fichier `.env`. |

---

## Étape 1 — Préparation du serveur

Mise à jour du système et installation des dépendances :

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install git nano curl -y
```

Installation de Docker et Docker Compose (script officiel) :

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

---

## Étape 2 — Téléchargement du projet

```bash
sudo mkdir -p /var/www
cd /var/www
sudo git clone https://github.com/ClientXCMS/clientxcms.git
cd clientxcms
```

---

## Étape 3 — Configuration du fichier `.env`

```bash
cp .env.example .env
nano .env
```

Renseignez les variables suivantes :

```env
APP_URL=http://192.168.50.139  # ← Votre IP ou domaine

# Correctif anti-erreur 419 (obligatoire en HTTP/IP locale)
SESSION_DRIVER=cookie
SESSION_SECURE_COOKIE=false
SESSION_SECURE=false

# Base de données
DB_CONNECTION=mysql
DB_HOST=database
DB_PORT=3306
DB_DATABASE=clientxcms
DB_NAME=clientxcms
DB_USERNAME=root
DB_PASSWORD=MettezUnMotDePasseFortIci123
```

> ⚠️ Remplacez `DB_PASSWORD` par un mot de passe fort de votre choix.

---

## Étape 4 — Lancement et initialisation

> Si vous réinstallez sur un serveur existant, purgez d'abord les anciens volumes :
> ```bash
> sudo docker compose down -v
> ```

**1. Déployez les conteneurs :**

```bash
sudo docker compose --env-file .env up -d
```

**2. Attendez ~30 secondes** que la base de données s'initialise.

**3. Générez la clé de chiffrement :**

```bash
sudo docker exec -it clientxcms-app-1 php artisan key:generate
```

**4. Videz le cache :**

```bash
sudo docker exec -it clientxcms-app-1 php artisan optimize:clear
```

**5. Créez le compte administrateur :**

```bash
sudo docker exec -it clientxcms-app-1 php artisan clientxcms:install-admin
```

---

## Étape 5 — Script de mise à jour du `.env`

Docker met la configuration en cache au démarrage. Ce script recharge proprement toute modification du `.env`.

**1. Créez le script :**

```bash
nano maj_env.sh
```

**2. Collez le contenu suivant :**

```bash
#!/bin/bash
echo "🔄 Application des modifications du fichier .env..."
sudo docker compose down
sudo docker compose --env-file .env up -d --force-recreate
echo "🧹 Nettoyage des caches Laravel..."
sudo docker exec -it clientxcms-app-1 php artisan optimize:clear
echo "✅ Configuration mise à jour avec succès !"
```

**3. Rendez-le exécutable :**

```bash
chmod +x maj_env.sh
```

**Utilisation :** après chaque modification du `.env`, exécutez simplement :

```bash
./maj_env.sh
```

---

## 🛠️ Résolution des problèmes

### ❌ Erreur 1045 — `Access denied` / Conteneur en boucle `Restarting`

**Cause :** le mot de passe dans `.env` a été modifié alors que la base de données avait déjà été initialisée avec un autre mot de passe.

**Solution :**

```bash
# 1. Éteignez tout et purgez le stockage corrompu
sudo docker compose down -v

# 2. Relancez proprement
./maj_env.sh
```

> ⏳ Attendez **45 secondes** avant de recréer le compte admin.

---

### ❌ Erreur 419 — `Page Expired` à la soumission d'un formulaire

**Cause :** le mode HTTPS strict est actif alors que vous naviguez sur une IP locale en HTTP — le navigateur bloque les cookies de session.

**Solution :** vérifiez que ces deux lignes sont bien présentes dans votre `.env` :

```env
SESSION_SECURE=false
SESSION_DRIVER=cookie
```

Puis rechargez la configuration :

```bash
./maj_env.sh
```

> 💡 Effectuez votre test dans un **onglet de navigation privée** pour éviter les conflits de cache navigateur.
