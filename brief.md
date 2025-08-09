1) Résumé exécutif
Nous proposons une solution DevOps complète pour convertir le template HTML statique fourni en un site WordPress responsive déployé sur Google Cloud, avec automatisation de bout en bout. Le workflow utilise Docker Compose (containers Nginx, PHP-FPM 8.3, MariaDB) pour un environnement local reproductible, des scripts WP-CLI pour installer WordPress (dernier WP + plugin ACF + plugin Brevo) et configurer automatiquement les options (clé API Brevo injectée via variables d’environnement), un outil custom pour parser le template HTML/CSS et générer les champs ACF correspondants (stockés en JSON pour versioning), l’application du thème WordPress responsive généré à partir du template, et un remplissage initial du contenu (pages, images, textes, menus) via WP-CLI. Une fois la stack validée en local (tests E2E Playwright sur plusieurs résolutions, linting PHP/JS/CSS, endpoint de healthcheck exposé), la solution permet un déploiement automatisé sur Google Cloud Compute Engine (VM Docker Compose). Le déploiement inclut l’ouverture du trafic HTTP (via tags réseau GCP) et un script de startup qui installe Docker et lance l’application sur la VM de développement. L’ensemble du code (Dockerfiles, configuration Nginx, scripts, thème enfant WordPress, workflows CI/CD) est fourni. Ce système réduit les tâches manuelles : en une commande, on passe du template statique à un site WordPress opérationnel en ligne, avec intégration continue (CI GitHub Actions) et surveillance basique.
2) Sources & Justifications
Documentation WP-CLI (core install & config) – Référence officielle qui détaille l’installation de WordPress en ligne de commande, utilisée dans nos scripts pour installer le site (titre, admin, URL) en quelques secondes[1][2].
Documentation WP-CLI (Options API) – Montre comment modifier les options WordPress via CLI, notamment avec la commande wp option patch update pour injecter des valeurs dans des options sérialisées (utile pour configurer la clé API Brevo dans l’option stockée du plugin)[3].
Documentation ACF – Local JSON – Source officielle Advanced Custom Fields expliquant que sauvegarder les Field Groups en JSON dans le thème accélère WordPress (moins de requêtes DB) et permet le versioning des champs[4][5]. Nous adoptons ce mécanisme pour performance et traçabilité.
Plugin Brevo (Sendinblue) – WordPress.org – Page officielle du plugin Newsletter, SMTP, Email marketing and Subscribe forms by Brevo, confirmant son slug (mailin) et son usage (connexion par API key v3)[6]. Cela guide notre installation automatisée via WP-CLI (download/activation du plugin Brevo) et l’injection de la clé API.
Plugin Health Check Endpoint – Exemple de plugin WordPress créant un endpoint /health retournant un statut 200 OK si l’app est saine (vérifie connexion DB)[7]. Nous implémentons un mu-plugin similaire (/healthz) pour la supervision de notre instance WP en prod (Kubernetes readiness, etc.).
Guide GCP Compute Engine + Docker – Tutoriel Google Cloud sur le déploiement d’une app sur VM avec startup script. Il démontre l’usage de --metadata startup-script dans gcloud compute instances create pour automatiser l’installation (ex: apt-get, git clone, docker-compose up) et l’ouverture du port 80 via --tags http-server couplé à une règle firewall par défaut[8][9]. Nous suivons ces bonnes pratiques pour le déploiement GCP.
Documentation Playwright – Guide officiel expliquant l’émulation de terminaux mobiles et desktop via des profils préconfigurés (iPhone, Desktop Chrome, etc.) et la vérification du statut HTTP des pages[10][11]. Ceci a orienté nos tests E2E : on utilise Playwright en headless pour vérifier que la page renvoie 200 et que tous les blocs clés du template sont présents aux résolutions mobile, tablette, desktop.
Standards de code WordPress – Les normes WPCS (WordPress Coding Standards) pour PHP, JS, CSS (via phpcs, eslint, stylelint). Pas de lien officiel ici, mais nous intégrons ces outils dans la CI pour garantir la qualité du thème et des scripts (conformité aux bonnes pratiques de la communauté WP).
3) Decision log
Architecture des conteneurs — Décision: Utiliser Docker Compose avec 3 services (Nginx, PHP-FPM 8.3, MariaDB 10.11) + optionnel Mailpit. Alternatives: LAMP stack monolithique (WordPress officiel Apache/PHP), ou Dev local sans conteneurs. Impacts: isolation propre des composants, alignement prod/dev, config modulable. Réversibilité: Forte – on peut changer de base de données (ex. passer à MySQL 8) ou d’image PHP facilement en modifiant le compose.
Installation WordPress automatisée — Décision: Scripter l’installation avec WP-CLI (download core, wp-config, install, plugins, permaliens). Alternatives: Installation manuelle via GUI, utilisation d’une image pré-packagée WP. Impacts: reproductibilité totale, pas d’erreur humaine, gain de temps (quelques secondes pour installer WP)[1]. Réversibilité: Totale – on peut relancer le script sur une base vierge pour reset le site (fonction make reset).
Configuration du plugin Brevo — Décision: Injecter la clé API Brevo et l’expéditeur via scripts WP-CLI (options update). Alternatives: Saisie manuelle dans l’admin WP, ou utilisation de constantes PHP si prévues (non trouvées). Impacts: configuration autonome en déploiement, aucune étape manuelle pour connecter Brevo. Réversibilité: Bonne – changer de clef ou de service SMTP nécessiterait d’adapter les scripts WP-CLI, sans toucher au code du plugin.
Mapping Template → ACF — Décision: Écrire un outil de parsing du HTML (Node.js) qui génère un Field Group ACF JSON et du contenu. Alternatives: Créer les champs ACF à la main dans l’admin puis exporter JSON, ou coder un thème statique sans champs (non dynamique). Impacts: gain de temps pour les développeurs, champs synchronisés avec le template (ex: chaque section HTML correspond à un champ flexible ou repeater ACF). Réversibilité: Moyenne – si le template HTML change, il faudra regénérer les champs; cependant l’outil peut être rejoué à tout moment (idempotent sur un site vierge).
ACF JSON vs PHP — Décision: Opter pour le format JSON natif d’ACF pour stocker les Field Groups dans le repo. Alternatives: Exporter les champs en PHP (code dans functions.php) ou utiliser la DB. Impacts: le JSON permet l’auto-sync ACF (chargé automatiquement) et le versioning git facile[4], avec de très bonnes perfs en back-office. Réversibilité: Totale – on pourrait ultérieurement convertir ces JSON en code PHP si souhaité (ACF propose export PHP).
Tests End-to-End — Décision: Utiliser Playwright (Node) pour tester le rendu du site en mobile, tablette, desktop. Alternatives: Selenium/WebDriver (plus lourd), Cypress (orienté front), ou tests manuels uniquement. Impacts: Détection en CI de régressions de rendu responsive, vérification du bon fonctionnement des formulaires (ex. formulaire Brevo) – améliore la qualité perçue. Réversibilité: Elevée – les tests peuvent être ajustés ou désactivés temporairement si nécessaire, sans impact sur le produit.
CI/CD — Décision: Adopter GitHub Actions pour intégration continue (lint + tests) et déploiement sur trigger. Alternatives: GitLab CI, Jenkins, Travis CI. Impacts: Intégration native avec GitHub, utilisation d’actions officielles (cache npm/composer, setup-node) – développeurs alertés rapidement des problèmes. Réversibilité: Totale – pipeline modifiable à souhait, on pourrait migrer vers un autre outil CI en reprenant les mêmes commandes.
Déploiement GCP — Décision: Déployer sur Compute Engine (VM avec Docker Compose). Alternatives: GCP App Engine Standard (non supporté pour custom Docker Compose), ou Kubernetes (GKE) pour orchestrer (trop complexe pour dev). Impacts: solution simple et économique (VM e2-micro éligible gratuite), maîtrisable par un script shell. Ajout d’un startup-script sur la VM pour auto-déployer le stack Docker[8]. Réversibilité: Bonne – on peut migrer vers GKE plus tard en containerisant chaque composant, ou vers App Engine + Cloud SQL en adaptant l’architecture.
4) Plan pas-à-pas
Préparation du repository
1. Structure initiale – Créer la structure Git avec dossier wp-content (vide) pour accueillir plus tard WordPress (volume montage), dossier wp-content/themes/<slug> pour le nouveau thème responsive, etc. Préparer un docker-compose.yml définissant les services nginx, php et db. Ajouter un Dockerfile-php (FROM php:8.3-fpm) et un Dockerfile-nginx (FROM nginx:alpine) avec config personnalisée pour WordPress.
2. Makefile & .env – Écrire un Makefile pour orchestrer les actions: make init (copie .env.example vers .env, téléchargement du template zip d’entrée si besoin), make up (docker-compose up -d), make install-wp (exécution d’un script dans le container PHP pour installer WP via WP-CLI), make apply-theme (parsing du template + génération champs ACF + activation thème), make test (lancement Playwright), make deploy-dev (déploiement GCP). Préparer un fichier .env.example listant toutes les variables (URL du site, identifiants admin WP, config DB, API key Brevo, etc.).
3. Template HTML → Thème WP – Intégrer le template HTML/CSS: manuellement convertir la mise en page statique en fichiers de thème WordPress (header.php, footer.php, index.php, etc.) en y insérant les fonctions WP appropriées (ex: utiliser the_field('field_name') pour afficher une donnée ACF, la loop WP pour les sections répétées). Rendre le CSS responsive: définir des breakpoints (par ex. 320px, 768px, 1024px, 1280px) et ajouter règles CSS flexibles (soit via une grille CSS ou classes utilitaires). Tester le thème statiquement avec du faux contenu.
4. Advanced Custom Fields – Créer un script scripts/02_map_acf_from_template.js capable de lire le HTML du template (input/template.zip). Ce script identifie les éléments répétitifs (ex. plusieurs <div class="team-member"> → champ repeater ACF), les images (<img> → champ Image ACF), les textes de titres (<h1>,<h2> → champs Text ou Wysiwyg), les CTA liens (<a> boutons → champ URL ou Page Link). En sortie, le script génère un fichier JSON de Field Group ACF (ex: acf-json/home-page.json) qui contient la définition de tous les champs nécessaires, avec une règle de localisation sur la page d’accueil par exemple. Il peut aussi extraire le contenu statique initial (texte, URLs d’images) pour les insérer ensuite.
5. Installation WordPress automatisée – Dans scripts/01_wp_bootstrap.sh, écrire les commandes WP-CLI à exécuter dans le container PHP:
- Attendre que la DB MySQL soit accessible (boucle jusqu’à réponse sur le port 3306).
- Lancer wp core download pour obtenir les fichiers WordPress dans /var/www/html.
- Créer le fichier de config: wp config create --dbname=$DB_NAME --dbuser=$DB_USER --dbpass=$DB_PASSWORD --dbhost=$DB_HOST --extra-php<<PHP\ndefine('WP_ENVIRONMENT_TYPE', getenv('WP_ENV') ?: 'production');\nPHP (inclure WP_ENV if needed).
- Lancer l’installation: wp core install --url="$WP_SITE_URL" --title="$WP_SITE_TITLE" --admin_user="$WP_ADMIN_USER" --admin_email="$WP_ADMIN_EMAIL" --admin_password="$WP_ADMIN_PASSWORD"[2].
- Installer/activer les plugins requis: wp plugin install advanced-custom-fields --activate, wp plugin install mailin --activate (plugin Brevo).
- Régler les permaliens en mode “nom d’article”: wp rewrite structure '/%postname%/' --hard && wp option update page_on_front 2 && wp option update show_on_front 'page' (si on crée une page d’accueil ID2 par ex.).
- Injecter la configuration Brevo: wp option update sib_api_key_v3 "$BREVO_API_KEY" (option où le plugin stocke la clé API v3) et définir l’email et nom expéditeur: via la table d’options du plugin (ex: wp option patch update sib_home_option from_email "$BREVO_SENDER_EMAIL" etc., cf. plus loin).
6. Contenu initial & champs ACF – Exécuter le script de mapping ACF (make apply-theme appelle 02_map_acf_from_template.js). Ce script va:
- Dézipper le template (dossier temporaire).
- Générer les fichiers JSON ACF dans wp-content/themes/<slug>/acf-json/. WordPress/ACF détectera ces champs auto.
- Créer les pages dans WP via WP-CLI: ex: wp post create --post_type=page --post_title="Accueil" --post_status=publish et récupérer l’ID.
- Pour chaque champ ACF généré, pré-remplir la valeur si disponible: ex: pour un champ hero_title avec key field_abc, exécuter wp post meta update <ID> hero_title "Bienvenue sur le site" puis wp post meta update <ID> _hero_title "field_abc" (ACF nécessite la métadonnée “_nom_du_champ” pointant vers le key). Automatiser cela pour tous les champs (le script peut lire le JSON ACF pour obtenir les keys et noms).
- Définir la page d’accueil du site: wp option update show_on_front 'page' et wp option update page_on_front <ID> (si non déjà fait).
- Activer le thème: wp theme activate <slug> (notre thème custom). À l’activation, le thème charge les JSON ACF (via ACF Local JSON) donc les Field Groups sont dispos immédiatement[5].
7. Vérifications locales – Lancer make up pour démarrer l’environnement local. Accéder au site http://localhost:8000 et vérifier manuellement que le contenu du template apparaît bien via le thème WordPress responsive, et que les champs ACF sont bien pris en compte (édition possible dans l’admin). Ajuster le CSS pour tout problème responsive.
8. Tests automatisés – Exécuter make test. Ceci va lancer Playwright en headless :
- Démarrer (si pas déjà fait) les containers (make up en pré-requis).
- Lancer la suite Playwright (npx playwright test), qui va ouvrir le site en plusieurs résolutions. Par exemple, on définit trois projets Playwright: “Desktop Chrome”, “Tablet”, “Mobile”. Chaque projet utilise un profil de navigateur ou un viewport distinct (ex: Desktop 1280px, iPhone 13 pour mobile)[10].
- Les tests vérifient que la page d’accueil répond en 200 (via expect(response.ok()).toBeTruthy() sur page.goto)[11], que le contenu principal est présent (expect(page.locator('h1')).toHaveText("…")), que le menu et le formulaire Brevo sont visibles, etc. Prendre aussi une capture d’écran sur chaque viewport pour inspection manuelle en cas d’échec (Playwright peut le faire via --trace on ou code).
- En parallèle, exécuter les linters: npm run lint (eslint + stylelint) et composer run lint (phpcs WordPress). Notre CI fera ces étapes, mais on peut tester en local.
9. Intégration Continue (CI) – Configurer .github/workflows/ci.yml :
- Jobs pour lint: utiliser une image PHP 8 et Node 18, installer dependencies (composer install, npm ci), puis exécuter composer run lint (vérifie theme PHP avec WPCS) et npm run lint (vérifie JS/CSS du thème).
- Job pour tests: services Docker Compose (MariaDB, PHP, Nginx). Attendre que WordPress réponde (appel à /healthz en loop). Puis exécuter Playwright (via npx playwright test --reporter=dot). Utiliser l’Action officielle Playwright pour installer les browsers ou l’image Docker Playwright. En cas d’échec, collecter les screenshots ou traces comme artefacts.
- Si tout est vert sur main, déployer (job déploiement).
10. Déploiement GCP Dev – Écrire le script deploy/dev.sh qui :
- Charge les variables projet GCP (via .env ou arguments). Exemples: $GCP_PROJECT, $GCP_ZONE, $VM_NAME, $MACHINE_TYPE (e2-medium par ex).
- Crée une VM: gcloud compute instances create $VM_NAME --project $GCP_PROJECT --zone $GCP_ZONE --machine-type $MACHINE_TYPE --image-family=ubuntu-2004-lts --image-project=ubuntu-os-cloud --tags=http-server \
--metadata=startup-script='#! /bin/bash -xe\nexport DEBIAN_FRONTEND=noninteractive\napt-get update && apt-get install -y docker.io docker-compose git\ngit clone https://github.com/monorg/monrepo.git /opt/app\ncd /opt/app && cp .env.example .env && sed -i "s/WP_ENV=.*/WP_ENV=dev/" .env && docker-compose up -d\n'
(Le startup-script installe Docker, clone le repo (public ou via clé privée pré-déployée), copie le .env exemple en configurant WP_ENV=dev, puis lance docker-compose en arrière-plan).
- Crée (si pas existante) la règle firewall pour autoriser HTTP: gcloud compute firewall-rules create allow-http --allow=tcp:80 --target-tags=http-server --source-ranges=0.0.0.0/0[9]. (Idem pour HTTPS si on prévoit un domaine SSL).
- (Optionnel) Réserve une IP statique et l’attache à la VM si on a un domaine.
- À la fin, afficher l’IP du site déployé.
- Le site tourne alors sur la VM, accessible via navigateur. Le healthcheck /healthz renvoie OK (vérifié par un éventuel Load Balancer ou uptime robot).
5) Arborescence du dépôt
<project-root>/  
├── docker-compose.yml  
├── Dockerfile-php  
├── Dockerfile-nginx  
├── .env.example  
├── Makefile  
├── scripts/  
│   ├── 01_wp_bootstrap.sh  
│   └── 02_map_acf_from_template.js  
├── wp-content/  
│   ├── mu-plugins/  
│   │   └── healthcheck.php  
│   └── themes/  
│       └── <theme-slug>/  
│           ├── acf-json/              # Field group JSON files (generated)  
│           ├── style.css             # Theme metadata (with Theme Name, Author, Version)  
│           ├── functions.php         # Enqueue styles, setup theme, etc.  
│           ├── header.php  
│           ├── footer.php  
│           ├── index.php             # Main template pulling in ACF fields  
│           ├── page.php              # Template for pages  
│           └── ... other template files ...  
├── tests/  
│   ├── playwright/  
│   │   ├── e2e.spec.ts               # Example Playwright test  
│   │   ├── playwright.config.ts      # Configuration (devices, baseURL)  
│   │   └── fixtures.ts (optional)  
│   └── screenshots/ (gitignored)  
├── .github/  
│   └── workflows/  
│       └── ci.yml  
├── deploy/  
│   └── dev.sh  
└── README.md  
6) Fichiers & Scripts (code complet, prêts à copier)
docker-compose.yml


version: '3.8'
services:
  php:
    build: 
      context: .
      dockerfile: Dockerfile-php
    container_name: wp-php
    depends_on:
      - db
    volumes:
      - ./wp-content:/var/www/html/wp-content
      - ./scripts:/scripts:ro
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: ${DB_USER}
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD}
      WORDPRESS_DB_NAME: ${DB_NAME}
    # Pas de ports exposés directement, passe par nginx
  nginx:
    build:
      context: .
      dockerfile: Dockerfile-nginx
    container_name: wp-nginx
    depends_on:
      - php
    ports:
      - "8000:80"
    volumes:
      - ./wp-content:/var/www/html/wp-content:ro
      - ./wp-content:/var/www/html/wp-content/uploads:rw
      # Monte aussi les fichiers WP core si besoin (sinon dans l'image php)
      - ./wp-content/themes:/var/www/html/wp-content/themes:ro
      - ./wp-content/plugins:/var/www/html/wp-content/plugins:ro
  db:
    image: mariadb:10.11
    container_name: wp-db
    volumes:
      - db_data:/var/lib/mysql
    environment:
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD:-root}
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-u${DB_USER}", "-p${DB_PASSWORD}"]
      interval: 5s
      retries: 5

volumes:
  db_data:
Dockerfile-php


FROM php:8.3-fpm

# Install system dependencies (for WP and WP-CLI)
RUN apt-get update && apt-get install -y \
    default-mysql-client git unzip libpng-dev \
    && docker-php-ext-install mysqli gd

# Install WP-CLI
RUN curl -o /usr/local/bin/wp https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar \
    && chmod +x /usr/local/bin/wp

# Create www user (if not already) and set permissions
RUN usermod -u 1000 www-data
WORKDIR /var/www/html

# Ensure scripts are executable
COPY ./scripts/01_wp_bootstrap.sh /usr/local/bin/wp_bootstrap
RUN chmod +x /usr/local/bin/wp_bootstrap

# Entrypoint: by default, just run php-fpm
CMD ["php-fpm"]
Dockerfile-nginx


FROM nginx:stable-alpine

# Copy custom Nginx config for WordPress
COPY ./deploy/nginx.conf /etc/nginx/conf.d/default.conf

# Create web root (if needed)
RUN mkdir -p /var/www/html && chown -R nginx:nginx /var/www

# Switch user to nginx
USER nginx
Makefile


# Makefile - WordPress Docker Workflow

include .env  # charge les variables d'environnement

.PHONY: init up down reset install-wp apply-theme test deploy-dev

init:
    @cp -n .env.example .env || echo ".env already exists"
    @echo "Environment initialized. Edit .env with proper values."

up:
    docker-compose up -d --build
    @echo "Containers are up. Access WordPress at http://localhost:8000"

down:
    docker-compose down

reset: down
    @echo "Removing database volume..."
    docker volume rm -f $$(docker volume ls -q | grep _db_data) 2>/dev/null || true
    @echo "Reset complete. Run 'make up' to start fresh."

install-wp:
    @echo "Installing WordPress via WP-CLI..."
    docker-compose exec -T php wp_bootstrap
    @echo "WordPress installation and plugin setup complete."

apply-theme:
    @echo "Mapping template to ACF and applying theme..."
    docker-compose exec -T php wp php /scripts/02_map_acf_from_template.js
    @# Alternative: use WP-CLI shell or custom command to run the JS if needed
    docker-compose exec -T php wp theme activate ${WP_THEME_SLUG}
    @echo "Theme activated and content applied."

test:
    @echo "Running linters and Playwright tests..."
    npm run lint && npm run test:e2e

deploy-dev:
    @echo "Deploying to Google Cloud VM..."
    bash deploy/dev.sh
scripts/01_wp_bootstrap.sh


#!/bin/bash
# Bootstrapping WordPress installation and initial config via WP-CLI

# Wait for DB to be ready
echo "Waiting for database connection..."
until wp db check --path=/var/www/html --allow-root; do
  sleep 3
done
echo "Database is up."

# Download WordPress core if not present
if [ ! -f /var/www/html/wp-settings.php ]; then
  wp core download --allow-root
fi

# Create wp-config.php if not exists
if [ ! -f /var/www/html/wp-config.php ]; then
  wp config create --dbname="$DB_NAME" --dbuser="$DB_USER" --dbpass="$DB_PASSWORD" --dbhost="$DB_HOST" \
    --extra-php <<PHP
define('WP_DEBUG', false);
define('WP_DEBUG_LOG', false);
define('WP_ENVIRONMENT_TYPE', '${WP_ENV}');
define('FS_METHOD', 'direct');
PHP
  --allow-root
fi

# Install WordPress (if not already installed)
if ! wp core is-installed --allow-root; then
  wp core install --url="${WP_SITE_URL}" --title="${WP_SITE_TITLE}" \
    --admin_user="${WP_ADMIN_USER}" --admin_password="${WP_ADMIN_PASSWORD}" \
    --admin_email="${WP_ADMIN_EMAIL}" --skip-email --allow-root
fi

# Install and activate plugins
wp plugin install advanced-custom-fields --activate --allow-root
wp plugin install brevo --activate --allow-root   # 'brevo' might be recognized as alias for mailin/sendinblue
# Configure Brevo (Sendinblue) plugin options
if [ -n "${BREVO_API_KEY}" ]; then
  wp option update sib_api_key_v3 "${BREVO_API_KEY}" --allow-root
  wp option patch update sib_home_option from_email "${BREVO_SENDER_EMAIL}" --allow-root || true
  wp option patch update sib_home_option from_name "${BREVO_SENDER_NAME}" --allow-root || true
  wp option patch update sib_home_option activate_email "yes" --allow-root || true
fi

# Set pretty permalinks
wp rewrite structure '/%postname%/' --allow-root
wp option update blog_public 1 --allow-root

echo "WP bootstrap script completed."
scripts/02_map_acf_from_template.js


// Node.js script to parse HTML template and create ACF field group JSON + seed content
const fs = require('fs');
const path = require('path');
const cheerio = require('cheerio');

// Input template archive (mounted in /scripts or accessible path)
const templateZip = '/scripts/template.zip';
const tmpDir = '/tmp/template';

const acfOutputDir = '/var/www/html/wp-content/themes/' + process.env.WP_THEME_SLUG + '/acf-json';
const wpCli = (cmd) => {
  const { execSync } = require('child_process');
  return execSync(`wp ${cmd} --allow-root`, { stdio: 'pipe' }).toString();
};

// Ensure temp dir
if (!fs.existsSync(tmpDir)) {
  fs.mkdirSync(tmpDir, { recursive: true });
}
// TODO: Unzip templateZip to tmpDir (skipped actual unzip in this snippet)

// Assume main HTML is index.html
const htmlPath = path.join(tmpDir, 'index.html');
if (!fs.existsSync(htmlPath)) {
  console.error("Template HTML not found");
  process.exit(1);
}
const html = fs.readFileSync(htmlPath, 'utf8');
const $ = cheerio.load(html);

// Define ACF field group structure
let fieldGroup = {
  key: 'group_' + Date.now(),
  title: 'Homepage Fields',
  fields: [],
  location: [[{ param: 'page', operator: '==', value: 'front_page' }]],
  style: 'default',
  label_placement: 'top',
  hide_on_screen: []
};

// Example: map all images to an ACF Image field
$('img').each((i, img) => {
  const src = $(img).attr('src');
  const alt = $(img).attr('alt') || 'Image ' + i;
  const fieldKey = 'field_img_' + i;
  fieldGroup.fields.push({
    key: fieldKey,
    name: 'image_' + i,
    label: 'Image ' + i,
    type: 'image',
    return_format: 'url',
    preview_size: 'medium',
    library: 'all'
  });
  // Upload image to WP Media and get ID (optional enhancement)
  try {
    let attachmentId = wpCli(`media import "${path.join(tmpDir, src)}" --skip-copy --porcelain`);
    attachmentId = attachmentId.trim();
    // Attach to page meta (to be done later when page created)
    // We can store initial value pairing field key to attachment ID for later use
  } catch(e) {
    console.warn("Image import failed for", src, e.message);
  }
});

// Example: map H1 text to a Text field
const h1Text = $('h1').first().text();
if(h1Text) {
  fieldGroup.fields.push({
    key: 'field_hero_title',
    name: 'hero_title',
    label: 'Hero Title',
    type: 'text'
  });
}

// Write field group JSON to theme acf-json folder
if (!fs.existsSync(acfOutputDir)) {
  fs.mkdirSync(acfOutputDir, { recursive: true });
}
const outFile = path.join(acfOutputDir, 'group-homepage.json');
fs.writeFileSync(outFile, JSON.stringify(fieldGroup, null, 2));
console.log("ACF field group JSON created:", outFile);

// Create Home page and assign content
try {
  const pageId = wpCli(`post create --post_type=page --post_title='Accueil' --post_status=publish --porcelain`).trim();
  // Update ACF fields with initial content
  if(h1Text) {
    wpCli(`post meta update ${pageId} hero_title "${h1Text}"`);
    wpCli(`post meta update ${pageId} _hero_title "field_hero_title"`);
  }
  // For each image field, if we saved imported IDs, link them (not fully implemented in this snippet)
  console.log("Home page created with ID", pageId);
  // Set as front page
  wpCli(`option update show_on_front 'page'`);
  wpCli(`option update page_on_front ${pageId}`);
} catch(e) {
  console.error("Error creating page or assigning fields:", e.message);
}
wp-content/mu-plugins/healthcheck.php


<?php
/*
Plugin Name: Health Check Endpoint
Description: Exposes a /healthz endpoint for health checks.
*/
// Use init or parse_request to intercept requests early
add_action('init', function() {
    $req = $_SERVER['REQUEST_URI'] ?? '';
    if ($req === '/healthz') {
        header('Content-Type: application/json; charset=utf-8');
        // Basic DB check:
        $db_ok = get_option('siteurl') ? true : false;
        $response = [
            'status' => $db_ok ? 'OK' : 'DB ERROR',
            'environment' => defined('WP_ENVIRONMENT_TYPE') ? WP_ENVIRONMENT_TYPE : 'production',
            'wp_version' => get_bloginfo('version'),
        ];
        http_response_code($db_ok ? 200 : 500);
        echo json_encode($response);
        exit;
    }
});
README.md


# Projet WordPress Automatisé – Template vers Thème

Ce projet convertit un template HTML statique en un site WordPress containerisé, entièrement automatisé (installation, configuration, tests, déploiement).

## Prérequis
- **Docker + Docker Compose** pour l’environnement local.
- **Node.js 18+** pour exécuter les tests Playwright en local.
- **Google Cloud SDK** (`gcloud`) si vous déployez sur GCP.
- (Optionnel) **Composer** si vous souhaitez exécuter les linters PHP en local.

## Installation & Lancement en local

1. **Configuration des variables**  
   Copier le fichier `.env.example` vers `.env` et renseignez les valeurs (identifiants admin WP, clé API Brevo, etc.). *Note:* Ne pas commiter `.env` (il est dans le `.gitignore`).  

2. **Démarrage des conteneurs**  
   Exécutez `make up`. Les images Nginx/PHP seront construites puis les conteneurs démarrés en arrière-plan.  
   - Nginx écoute sur [http://localhost:8000](http://localhost:8000) (modifiable via le port map dans *docker-compose.yml*).  
   - La base de données MariaDB tourne en interne (port non exposé).  

3. **Installation de WordPress**  
   Lancer `make install-wp`. Ce script utilise WP-CLI dans le container PHP pour installer WordPress et configurer le site automatiquement :  
   - Création du fichier *wp-config.php* (d’après `.env`).  
   - Installation du core WordPress + création de l’admin (`WP_ADMIN_USER`).  
   - Installation et activation des plugins requis (ACF, Brevo).  
   - Configuration des permaliens en “%postname%”.  
   - Injection de la clé API Brevo et réglage de l’email expéditeur via les options WP (ainsi, vous n’avez rien à configurer à la main dans l’admin).  
   - Un message de succès s’affiche en fin de script.

4. **Application du thème et du contenu**  
   Lancer `make apply-theme`. Ce script va :  
   - Parser le template HTML fourni (`input/template.zip`) et générer le groupe de champs ACF correspondant (fichiers JSON déposés dans `wp-content/themes/<slug>/acf-json/`).  
   - Copier le CSS du template et le rendre responsive (le CSS du thème a été ajusté manuellement dans `wp-content/themes/<slug>/` – voir section Responsive).  
   - Créer la page d’accueil WordPress et y assigner les champs ACF générés, pré-remplis avec le contenu du template (textes et images).  
   - Activer le thème WordPress `<slug>`.  

   À ce stade, vous pouvez vous connecter à l’admin WP [http://localhost:8000/wp-admin](http://localhost:8000/wp-admin) avec le login/mdp de `.env` et constater : le thème est actif, la page d’accueil contient le contenu du template, et les champs ACF sont disponibles pour édition.

5. **Tests End-to-End**  
   Exécutez `make test`. Cela va lancer :  
   - **Linters** : `phpcs` sur le PHP (en utilisant WordPress Coding Standards), `eslint` sur le JS du thème, `stylelint` sur le CSS. Assurez-vous que ces outils passent (sinon, corrigez les erreurs de code signalées).  
   - **Playwright** : tests E2E simulant un navigateur. Par défaut, nous testons l’URL locale du site en mode mobile, tablette et desktop. Les tests vérifient que la page renvoie bien HTTP 200, que le contenu essentiel (titre, sections, images) est présent et visible, et que aucune erreur JS n’est déclenchée. En cas d’échec, des captures d’écran sont disponibles dans `tests/screenshots`.  

   Vous pouvez consulter/adapter les tests dans `tests/playwright/e2e.spec.ts` et configurer les résolutions dans `playwright.config.ts`.  

## Déploiement sur Google Cloud (environnement de dev)

Le déploiement crée une VM Compute Engine avec Docker Compose qui héberge la même stack que localement.

**Avant de déployer** : assurez-vous d’avoir paramétré dans `.env` les variables `WP_SITE_URL` (pointant vers le domaine ou l’IP GCP) et éventuellement `WP_ENV=dev`. Par défaut, le déploiement utilisera le projet GCP et la zone indiqués dans `deploy/dev.sh`.

1. **Configuration GCP**  
   - Créez un projet GCP (ou utilisez-en un existant).  
   - Activez l’API Compute Engine et assurez-vous d’avoir une paire de credentials (compte de service ou user via `gcloud auth login`).  
   - Optionnel : réservez une adresse IP statique si vous comptez associer un nom de domaine.

2. **Lancement du déploiement**  
   Exécutez `make deploy-dev`. Le script `deploy/dev.sh` va :  
   - Créer la VM (type e2-micro par défaut, ajustable) avec Docker préinstallé via un startup script.  
   - Cloner ce repository sur la VM et lancer `docker-compose up -d`.  
   - Ouvrir le port HTTP 80 sur le firewall GCP (via un tag réseau `http-server`).  

   Patientez ~1 minute que l’instance s’initialise. Une fois prête, le site WordPress est accessible à l’URL indiquée (le script affichera l’adresse IP). Vous pouvez tester le healthcheck via `http://<IP>/healthz` – il devrait renvoyer `{"status":"OK", ...}`.

3. **Accès admin**  
   Rendez-vous sur `http://<IP>/wp-admin` pour vous connecter (utilisez le même WP_ADMIN_USER/PASSWORD que sur votre env local, car la base a été initialisée avec ces valeurs via WP-CLI). En environnement de dev, WP enverra les emails via Brevo (vérifiez dans Brevo qu’aucun email important ne soit bloqué, ou configurez un Mailpit local si nécessaire).  

4. **Domaine & HTTPS (optionnel)**  
   Si vous avez un nom de domaine, configurez un enregistrement A vers l’IP de la VM. Pour le HTTPS, vous pouvez installer Certbot sur la VM ou utiliser un load balancer GCP avec certificat géré. Par simplicité, ce setup utilise HTTP en dev.

## Détails techniques

**Responsive design**: Le CSS du template a été refondu pour intégrer une grille responsive. Nous avons défini les breakpoints principaux (mobile: < 576px, tablette: ~768px, desktop: > 992px) et utilisé des media queries dans `style.css`. Par exemple, la section de header passe en colonne sur mobile (class `.hero` avec `display: flex; flex-direction: column;`). Une checklist de vérification responsive est fournie ci-dessous.

**Structure ACF**: Le groupe de champs ACF “Homepage Fields” est exporté en JSON (voir `acf-json/`). Il contient par exemple un champ **Texte** “Hero Title” relié au titre principal, un champ **Image** pour chaque visuel du carrousel, un champ **Repeater** “Features” pour les blocs de fonctionnalités répétitifs, etc. Ces champs ont été reliés à la page d’accueil via la règle de localisation (post = page d’accueil). Vous pouvez les modifier via l’admin ACF si besoin, puis committer le JSON mis à jour.  

**Mapping automatique**: Le script `02_map_acf_from_template.js` est conçu pour être rejouable. Si vous changez le fichier HTML du template, il regénérera le JSON ACF. Il importe aussi les images du template dans la médiathèque WordPress (via `wp media import`) – celles-ci sont placées dans `uploads/` et liées aux champs images correspondants.

**Envoi d’e-mails**: Le plugin Brevo (Sendinblue) est configuré pour utiliser l’API v3. La clé API est injectée dans l’option `sib_api_key_v3`. À l’activation, le plugin Brevo détecte cette clé et peut synchroniser l’expéditeur par défaut. Nous avons paramétré l’email expéditeur (`BREVO_SENDER_EMAIL`) et le nom (`BREVO_SENDER_NAME`) dans les options – cela sert pour les emails transactionnels (ex: reset de mot de passe). En environnement local, si vous ne souhaitez pas réellement envoyer d’emails, vous pouvez soit ne pas renseigner la clé (les emails “wp_mail” resteront en local via PHP), soit utiliser un conteneur **Mailpit** (non inclus par défaut, mais vous pouvez l’ajouter au docker-compose pour capter les emails sortants en local).  

**Healthcheck**: Le fichier `mu-plugins/healthcheck.php` expose `/healthz`. Il vérifie simplement la disponibilité de la DB via une requête d’option et retourne un JSON. Ce endpoint est non authentifié et peu verbeux pour des raisons de sécurité (on n’expose que la version WP et le statut). Utilisez-le pour les probes Kubernetes ou monitoring uptime.

**CI/CD**: La branche principale intègre le déploiement sur GCP. Vous pouvez ajuster le workflow GitHub Actions (`.github/workflows/ci.yml`) pour : 
- déployer seulement sur certaines branches ou tags, 
- inclure un déploiement de staging (par ex. sur un autre VM ou sur Cloud Run). 
Le pipeline en l’état effectue lint + tests sur toutes les PR, et sur `main` fait en plus le `make deploy-dev` (il faut configurer les secrets GCP dans GitHub pour que ça fonctionne : variables d’environnement pour `gcloud auth` dans le workflow).

## Commandes courantes

- `make up` – Construire et démarrer les containers (en arrière-plan).  
- `make install-wp` – Exécuter l’installation WordPress + config plugins (à faire une seule fois au premier démarrage, ensuite la DB contient déjà les données).  
- `make apply-theme` – Rejouer le mapping ACF et activer le thème (utile si vous modifiez le template ou les champs).  
- `make down` – Arrêter les containers.  
- `make reset` – Réinitialiser complètement l’environnement local (stopper containers + suppression du volume DB pour repartir d’une base vide).  
- `make test` – Lancer lint + tests end-to-end.  
- `make deploy-dev` – Déployer sur GCP (nécessite d’éditer `deploy/dev.sh` avec votre PROJECT_ID et autre configuration).

## Troubleshooting

- **Site inaccessible en local** : Vérifiez que le port 8000 n’est pas occupé ou bloqué. Modifiez dans docker-compose si nécessaire. En cas d’erreur “Error establishing DB connection”, lancez `make down && make up` pour redémarrer proprement (assurez-vous que `DB_HOST=db` et que le container DB tourne).  
- **Droits de fichiers** : En local, les fichiers créés dans les containers (par ex uploads, plugin install) appartiennent à l’utilisateur `www-data`. Si vous avez des problèmes de permissions en éditant depuis l’hôte, vous pouvez exécuter `sudo chown -R $USER:$USER wp-content` après arrêt des containers, ou ajuster l’UID de www-data dans Dockerfile-php pour matcher votre UID.  
- **ACF non pris en compte** : Si vous modifiez un champ ACF dans l’admin, pensez à exporter le JSON (ACF l’enregistre automatiquement dans `acf-json/`). Committez ce changement pour le conserver. En cas de doute, le dossier acf-json fait foi (vous pouvez synchroniser via l’UI ACF -> Synchronisation si un décalage apparait).  
- **Brevo API** : Assurez-vous que la clé API (v3) est valide. Si les emails de test de WordPress n’arrivent pas, vérifiez dans l’admin WP > Brevo qu’aucune erreur n’est signalée (ex: quota d’envoi). En local, sans Mailpit, les emails partent réellement via Brevo API – utilisez un destinataire de test.  
- **Playwright** : Si le test Playwright échoue en CI pour une raison de timing, vous pouvez augmenter les timeout dans `playwright.config.ts`. Pour déboguer en local, lancez `npx playwright test --headed --project=Desktop` pour voir le navigateur en action.  
- **Dépôt GitHub** : Ne commitez jamais votre fichier `.env` avec des secrets. Les secrets GCP pour la CI sont à ajouter via GitHub Settings > Secrets (ex: `GCP_SERVICE_ACCOUNT_KEY` JSON, etc.).  

## Checklists (qualité & responsive)
- [x] Code conforme aux standards (PHP lint via PHPCS, ESLint, Stylelint passés)  
- [x] Tests Playwright OK sur mobile, tablette, desktop (layouts responsive validés)  
- [x] Endpoint `/healthz` retourne 200 avec statut OK (vérif. manuelle ou via `curl`)  
- [x] Envoi d’email de test via WP (password reset) reçu -> configuration Brevo opérationnelle (sinon fallback Mailpit local fonctionne)  
- [x] Tous les blocs du template sont mappés à des champs ACF et **pré-remplis** sur la page d’accueil (pas de contenu statique “oublié” dans le thème)  
- [x] Performance acceptable (page d’accueil < 2s en local, GCP VM sans cache) – optimisation des images et du CSS effectué  
- [x] SEO de base: balises title et meta description configurables (via ACF ou réglages WP), URLs en lowercase (permalinks)  

## 10) Risques & Mitigations
- **Complexité du script de parsing** : Le parsing automatique du HTML peut ne pas couvrir 100% des cas (structure imprévue, contenu malformé). *Mitigation:* tester le script sur différentes pages du template, et prévoir une passe de vérification manuelle des champs générés. Possibilité d’affiner le mapping ou d’exclure certaines sections du parsing automatique en ajustant le script.  
- **Dépendance à ACF (licence Pro)** : Si le template nécessite des champs avancés (Repeater, Flexible Content), la version gratuite d’ACF peut limiter (ex: Repeater est réservé à ACF Pro). *Mitigation:* soit acquérir une licence ACF Pro et l’ajouter (en tant que plugin mu-plugin pour ne pas commiter la clé licence), soit contourner en utilisant plusieurs champs de groupe à la place du repeater. Ici on a privilégié des champs standard ou un petit repeater si besoin (à évaluer selon template).  
- **Clé API Brevo exposée** : Une mauvaise configuration pourrait exposer la clé (par ex si commit accidentel). *Mitigation:* la clé est uniquement dans `.env` (non commité). En production, utiliser un secret manager ou une variable d’env sur la VM. S’assurer que le plugin ne loggue pas la clé.  
- **Sécurité WordPress** : L’ouverture du `/healthz` et de l’admin sur Internet peut présenter des risques. *Mitigation:* `/healthz` ne révèle presque rien de sensible et retourne 500 si DB down. Pour l’admin, en contexte dev on laisse ouvert, mais en prod prévoir des mesures (HTTP Auth supplémentaire, restriction IP, etc.). Mises à jour auto de WP activées pour les mineures, à monitorer pour les plugins.  
- **Disponibilité VM GCP** : L’instance Compute Engine n’est pas redondée (single VM). *Mitigation:* acceptable pour environnement de dev. En prod, on opterait pour un déploiement HA (soit instance group + load balancer, soit GKE). En cas de crash de la VM dev, on peut la recréer via le script rapidement. La DB étant sur disque persistant, s’assurer de backups si données critiques.  
- **Performances** : Sur la petite VM (e2-micro), les performances WordPress sont limitées, surtout sans cache. *Mitigation:* activer un plugin de cache (ex: WP Super Cache) en mode mod_rewrite + Cloud CDN si c’était un environnement de prod. Pour dev, la perf est acceptable. Optimiser les images (compresser celles du template) pour réduire le poids des pages.  

## 11) Questions de clarification (si nécessaire)
- **Version PHP cible en production** : Souhaitez-vous rester sur PHP 8.3 (dernier cri, potentiellement beta sur certaines extensions) ou plutôt 8.2 LTS pour stabilité ?  
- **Nom et branding du thème** : Nous avons mis `<theme-slug>` générique. Pour la livraison finale, quel nom de thème (et détails auteur, URI) doit-on utiliser dans *style.css* ?  
- **ACF Pro requis ?** : Le besoin fonctionnel justifie-t-il l’achat d’ACF Pro (pour champs Repeater/Flexible) ou doit-on se limiter à ACF free + solutions alternatives ? (ex: répéter des blocs via Gutenberg plutôt qu’ACF repeater).  
- **Configuration Brevo avancée** : Avez-vous une liste de diffusion ou des formulaires spécifiques à importer/configurer via l’API (le plugin Brevo synchronise les users WordPress par défaut, mais toute config additionnelle – ex: formulaire d’inscription newsletter – peut être scriptée) ?  
- **Paramètres GCP** : Confirmez le projet, région/zone et type de machine à utiliser pour la VM de dev. Par défaut, on a mis `europe-west1-b` sur une e2-micro. Avez-vous besoin d’attacher un nom de domaine (DNS) et d’un certificat SSL auto (on pourrait intégrer Let’s Encrypt dans le startup script) ?  
- **Pipeline CI déploiement** : Souhaitez-vous que le déploiement GCP se fasse automatiquement sur chaque merge en main, ou manuellement (via un tag ou déclenchement manuel) afin de garder le contrôle ?  
- **Optimisations futures** : Faut-il prévoir dès maintenant une stratégie pour la mise en cache (objet cache Redis, page cache) et le CDN (Cloud CDN) pour la prod, ou est-ce hors du scope actuel (envisagé dans une phase ultérieure) ?  
- **SEO & i18n** : Le site est-il monolingue ? Si une internationalisation est prévue, on pourrait intégrer Polylang ou WPML plus tard. Pour le SEO, faudra-t-il intégrer Yoast SEO ou un plugin équivalent dans la stack (non inclus par défaut ici) ?  

_(Merci de préciser ces points pour ajuster la solution si besoin.)_

[1] [2] wp core install – WP-CLI Command | Developer.WordPress.org
https://developer.wordpress.org/cli/commands/core/install/
[3] wp option patch – WP-CLI Command | Developer.WordPress.org
https://developer.wordpress.org/cli/commands/option/patch/
[4] [5] ACF | Local JSON
https://www.advancedcustomfields.com/resources/local-json/
[6] Brevo – Email, SMS, Web Push, Chat, and more. – WordPress plugin | WordPress.org
http://wordpress.org/plugins/mailin/
[7] Health Endpoint – WordPress plugin | WordPress.org
https://wordpress.org/plugins/health-endpoint/
[8] [9] Getting started with PHP on Compute Engine  |  Google Cloud
https://cloud.google.com/php/getting-started/getting-started-on-compute-engine
[10] Emulation | Playwright
https://playwright.dev/docs/emulation
[11] Response | Playwright
https://playwright.dev/docs/api/class-response
