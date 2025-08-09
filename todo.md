# TODO - Configuration Machine pour Agent Executor

## 📋 Prérequis Logiciels à Installer

### 1. Docker & Docker Compose ⚠️ CRITIQUE
```bash
# Windows (via Docker Desktop)
- Installer Docker Desktop for Windows
- Vérifier: docker --version && docker-compose --version
- S'assurer que Docker Desktop est démarré
```

### 2. Node.js 18+ ⚠️ CRITIQUE
```bash
# Windows (via Node.js installer ou nvm)
- Installer Node.js v18 ou supérieur
- Vérifier: node --version && npm --version
- npm install -g npx (si pas déjà présent)
```

### 3. Google Cloud SDK (pour déploiement GCP) 🔧 OPTIONNEL
```bash
# Windows
- Installer Google Cloud SDK
- Vérifier: gcloud --version
- Authentification: gcloud auth login
```

### 4. Git 🔧 REQUIS
```bash
# Windows (généralement déjà présent)
- Vérifier: git --version
```

## 🔧 Variables d'Environnement à Définir

### 1. Copier et configurer .env
```bash
# À exécuter dans le répertoire du projet
cp .env.example .env
```

### 2. Variables CRITIQUES à renseigner dans .env

#### WordPress (REQUIS)
```makefile
WP_SITE_URL=http://localhost:8000
WP_SITE_TITLE=Site WordPress Responsive
WP_ADMIN_USER=admin
WP_ADMIN_EMAIL=admin@example.com
WP_ADMIN_PASSWORD=admin123  # ⚠️ Changer en production
WP_ENV=local
```

#### Base de données (REQUIS)
```makefile
DB_NAME=wordpress
DB_USER=wp
DB_PASSWORD=wp
DB_HOST=db
DB_ROOT_PASSWORD=root
```

#### Thème (REQUIS)
```makefile
WP_THEME_NAME=Auto Theme
WP_THEME_SLUG=auto-theme
WP_THEME_AUTHOR=Sooatek
```

#### Brevo (OPTIONNEL pour développement local)
```makefile
BREVO_API_KEY=                    # Laisser vide pour dev local
BREVO_SENDER_EMAIL=no-reply@example.com
BREVO_SENDER_NAME=Website
```

#### GCP (REQUIS pour déploiement seulement)
```makefile
GCP_PROJECT_ID=                   # ID du projet Google Cloud
GCP_ZONE=europe-west1-b
GCP_VM_NAME=wp-auto-dev
GCP_MACHINE_TYPE=e2-micro
GCP_REPO_GIT=https://github.com/<org>/<repo>.git
```

## 📁 Structure de Fichiers Requis

### 1. Créer le dossier input
```bash
mkdir input
# Le fichier template.zip sera placé ici par l'utilisateur
```

### 2. Vérifier la structure attendue
```
projet/
├─ .env                  # ⚠️ À créer depuis .env.example
├─ input/
│  └─ template.zip       # ⚠️ Template HTML à fournir
├─ docker-compose.yml
├─ Makefile
├─ package.json
└─ wp-content/themes/auto-theme/
```

## ⚡ Tests de Validation Environnement

### 1. Test Docker
```bash
docker --version
docker-compose --version
docker run hello-world
```

### 2. Test Node.js
```bash
node --version  # Doit être >= 18.x
npm --version
```

### 3. Test Variables d'environnement
```bash
# Vérifier que .env existe et contient les bonnes variables
cat .env | grep -E "WP_SITE_URL|DB_NAME|WP_THEME_SLUG"
```

## 🚀 Commandes de Démarrage pour Agent Executor

### 1. Initialisation Projet
```bash
make init        # Copie .env.example → .env
```

### 2. Démarrage Stack Docker
```bash
make up          # Build et start containers Docker
```

### 3. Installation WordPress
```bash
make install-wp  # Install WP + plugins + config Brevo
```

### 4. Application du thème (nécessite template.zip dans input/)
```bash
make apply-theme # Parse template → génère ACF → active thème
```

### 5. Tests
```bash
make test        # Lint + tests Playwright E2E
```

## ⚠️ Points d'Attention pour l'Agent

### 1. Dépendances Docker
- L'agent doit s'assurer que Docker Desktop est démarré avant d'exécuter make up
- Vérifier que les ports 8000 ne sont pas occupés

### 2. Template HTML
- Le fichier input/template.zip doit être fourni par l'utilisateur
- L'agent doit vérifier son existence avant make apply-theme

### 3. Permissions Windows
- S'assurer que Docker a accès aux volumes montés
- Vérifier les permissions sur le dossier wp-content/

### 4. Variables sensibles
- Ne jamais commiter le fichier .env
- Utiliser des mots de passe robustes en production

## 🔍 Commandes de Diagnostic

### 1. Vérification Stack
```bash
docker-compose ps                    # Status des containers
curl http://localhost:8000/healthz   # Test endpoint santé
```

### 2. Logs Debug
```bash
docker-compose logs php    # Logs PHP/WordPress
docker-compose logs nginx  # Logs serveur web
```

### 3. Accès containers
```bash
docker-compose exec php bash        # Shell container PHP
docker-compose exec php wp --info   # Info WP-CLI
```

## ✅ Checklist Finale

- [ ] Docker & Docker Compose installés et fonctionnels
- [ ] Node.js 18+ installé
- [ ] Fichier .env créé et configuré
- [ ] Dossier input/ créé
- [ ] Template HTML disponible dans input/template.zip
- [ ] Port 8000 libre
- [ ] Docker Desktop démarré (Windows)
- [ ] Variables GCP configurées (si déploiement prévu)

## 🎯 Commande de Test Complet

```bash
# Test rapide de l'environnement complet
make init && make up && make install-wp
# Puis placer template.zip dans input/ et lancer:
# make apply-theme && make test
```

**L'agent peut maintenant exécuter le workflow complet de transformation HTML → WordPress !**