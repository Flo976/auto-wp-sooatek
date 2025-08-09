# html-to-wp-generator (seed repo)

> Dépôt prérempli prêt pour l’Agent d’Exécution. Contient la stack Docker, scripts WP-CLI, thème WP skeleton, tests Playwright, CI GitHub Actions et déploiement GCP. Remplace les placeholders dans `.env` puis lance `make up && make install-wp && make apply-theme`.

---

## Fichiers & dossiers

```text
.
├─ .editorconfig
├─ .gitignore
├─ .env.example
├─ README.md
├─ Makefile
├─ docker-compose.yml
├─ Dockerfile-php
├─ Dockerfile-nginx
├─ deploy/
│  ├─ dev.sh
│  └─ nginx.conf
├─ scripts/
│  ├─ 01_wp_bootstrap.sh
│  └─ 02_map_acf_from_template.js
├─ input/
│  └─ template.zip          # à fournir par l’utilisateur (non commité)
├─ wp-content/
│  ├─ mu-plugins/
│  │  └─ healthcheck.php
│  └─ themes/
│     └─ auto-theme/
│        ├─ acf-json/.gitkeep
│        ├─ style.css
│        ├─ functions.php
│        ├─ header.php
│        ├─ footer.php
│        ├─ index.php
│        └─ page.php
├─ tests/
│  └─ playwright/
│     ├─ e2e.spec.ts
│     └─ playwright.config.ts
├─ .github/
│  └─ workflows/
│     └─ ci.yml
├─ package.json
├─ package-lock.json        # (facultatif pour seed)
└─ composer.json            # pour phpcs/WPCS (lint PHP)
```

---

### `.editorconfig`
```ini
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
indent_style = space
indent_size = 2
```

### `.gitignore`
```gitignore
# env & secrets
.env
.env.*

# node
node_modules/
.npm/

# composer
vendor/
composer.lock

# logs & caches
*.log
.cache/

# playwright outputs
playwright-report/
playwright/.cache/
/test-results/
/tests/screenshots/

# uploads & db volumes (local only)
wp-content/uploads/
.docker/
```

### `.env.example`
```makefile
# ---- WordPress ----
WP_SITE_URL=http://localhost:8000
WP_SITE_TITLE=Site WordPress Responsive
WP_ADMIN_USER=admin
WP_ADMIN_EMAIL=admin@example.com
WP_ADMIN_PASSWORD=admin123
WP_ENV=local

# ---- Base de données ----
DB_NAME=wordpress
DB_USER=wp
DB_PASSWORD=wp
DB_HOST=db
DB_ROOT_PASSWORD=root

# ---- Thème ----
WP_THEME_NAME=Auto Theme
WP_THEME_SLUG=auto-theme
WP_THEME_AUTHOR=Sooatek

# ---- Brevo ----
BREVO_API_KEY=
BREVO_SENDER_EMAIL=no-reply@example.com
BREVO_SENDER_NAME=Website

# ---- GCP (dev) ----
GCP_PROJECT_ID=
GCP_ZONE=europe-west1-b
GCP_VM_NAME=wp-auto-dev
GCP_MACHINE_TYPE=e2-micro
GCP_REPO_GIT=https://github.com/<org>/<repo>.git
```

### `README.md`
```md
# html-to-wp-generator (seed)

Automatisation de la conversion d’un template HTML/CSS en thème WordPress responsive avec ACF, config Brevo, tests Playwright et déploiement GCP.

## Démarrage rapide
```bash
cp .env.example .env
# Éditez .env si besoin
make up
make install-wp
# Placez votre template dans input/template.zip
make apply-theme
make test
```
Le site est servi sur **http://localhost:8000**.

## Commandes Makefile
- `make up` : build + start Docker
- `make down` : stop
- `make reset` : reset DB (supprime volume DB)
- `make install-wp` : installe WordPress + plugins + configuration Brevo
- `make apply-theme` : parse template, génère ACF JSON, active le thème, seed contenu
- `make test` : lint + tests E2E Playwright
- `make deploy-dev` : déploie sur GCP Compute Engine

## Prérequis
- Docker + Docker Compose
- Node 18+
- (CI) GitHub Actions
- (Déploiement) Google Cloud SDK si exécution locale de `deploy/dev.sh`
```

### `Makefile`
```make
include .env

.PHONY: init up down reset install-wp apply-theme test deploy-dev node-ci

init:
	@cp -n .env.example .env || true
	@echo "Init done. Edit .env if needed."

up:
	docker-compose up -d --build
	@echo "Open: http://localhost:8000"

down:
	docker-compose down

reset: down
	-@docker volume rm $$(docker volume ls -q | grep _db_data) 2>/dev/null || true
	@echo "DB volume removed. Run make up again."

install-wp:
	docker-compose exec -T php bash /usr/local/bin/wp_bootstrap

apply-theme:
	docker-compose run --rm node npm ci
	docker-compose run --rm node node /work/scripts/02_map_acf_from_template.js --theme $(WP_THEME_SLUG)
	docker-compose exec -T php wp theme activate $(WP_THEME_SLUG) --allow-root

node-ci:
	docker-compose run --rm node npm ci

test: node-ci
	docker-compose run --rm node npm run lint
	docker-compose run --rm node npm run test:e2e

deploy-dev:
	bash deploy/dev.sh
```

### `docker-compose.yml`
```yaml
version: '3.8'

services:
  db:
    image: mariadb:10.11
    environment:
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql

  php:
    build:
      context: .
      dockerfile: Dockerfile-php
    environment:
      DB_NAME: ${DB_NAME}
      DB_USER: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}
      DB_HOST: db
      WP_SITE_URL: ${WP_SITE_URL}
      WP_SITE_TITLE: ${WP_SITE_TITLE}
      WP_ADMIN_USER: ${WP_ADMIN_USER}
      WP_ADMIN_EMAIL: ${WP_ADMIN_EMAIL}
      WP_ADMIN_PASSWORD: ${WP_ADMIN_PASSWORD}
      WP_ENV: ${WP_ENV}
      BREVO_API_KEY: ${BREVO_API_KEY}
      BREVO_SENDER_EMAIL: ${BREVO_SENDER_EMAIL}
      BREVO_SENDER_NAME: ${BREVO_SENDER_NAME}
      WP_THEME_SLUG: ${WP_THEME_SLUG}
    volumes:
      - ./wp-content:/var/www/html/wp-content
      - ./scripts:/scripts
      - ./input:/input

  nginx:
    build:
      context: .
      dockerfile: Dockerfile-nginx
    ports:
      - "8000:80"
    depends_on:
      - php
    volumes:
      - ./wp-content:/var/www/html/wp-content:ro

  node:
    image: mcr.microsoft.com/playwright:v1.47.0-jammy
    working_dir: /work
    volumes:
      - ./:/work
    entrypoint: ["bash","-lc"]

volumes:
  db_data:
```

### `Dockerfile-php`
```dockerfile
FROM php:8.3-fpm

RUN apt-get update && apt-get install -y \
    default-mysql-client git unzip libpng-dev libjpeg62-turbo-dev libfreetype6-dev \
  && docker-php-ext-configure gd --with-freetype --with-jpeg \
  && docker-php-ext-install gd mysqli \
  && curl -o /usr/local/bin/wp https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar \
  && chmod +x /usr/local/bin/wp \
  && mkdir -p /var/www/html

WORKDIR /var/www/html

COPY scripts/01_wp_bootstrap.sh /usr/local/bin/wp_bootstrap
RUN chmod +x /usr/local/bin/wp_bootstrap

CMD ["php-fpm"]
```

### `Dockerfile-nginx`
```dockerfile
FROM nginx:stable-alpine
COPY deploy/nginx.conf /etc/nginx/conf.d/default.conf
RUN mkdir -p /var/www/html && chown -R nginx:nginx /var/www/html
```

### `deploy/nginx.conf`
```nginx
server {
  listen 80;
  server_name _;

  root /var/www/html;
  index index.php index.html;

  location /healthz { try_files $uri =404; }

  location / {
    try_files $uri $uri/ /index.php?$args;
  }

  location ~ \.php$ {
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_pass php:9000;
    fastcgi_read_timeout 120s;
  }

  location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ { expires max; log_not_found off; }
}
```

### `deploy/dev.sh`
```bash
#!/usr/bin/env bash
set -euo pipefail

: "${GCP_PROJECT_ID:?set in .env}"
: "${GCP_ZONE:=europe-west1-b}"
: "${GCP_VM_NAME:=wp-auto-dev}"
: "${GCP_MACHINE_TYPE:=e2-micro}"
: "${GCP_REPO_GIT:?https repo to clone}"

startup_script='#!/bin/bash -xe
apt-get update && apt-get install -y docker.io docker-compose git
mkdir -p /opt/app && cd /opt/app
git clone '"$GCP_REPO_GIT"' .
cp .env.example .env
sed -i "s/WP_ENV=.*/WP_ENV=dev/" .env
/usr/bin/docker-compose up -d --build
'

gcloud config set project "$GCP_PROJECT_ID"
gcloud compute instances create "$GCP_VM_NAME" \
  --zone "$GCP_ZONE" \
  --machine-type "$GCP_MACHINE_TYPE" \
  --image-family=ubuntu-2204-lts --image-project=ubuntu-os-cloud \
  --tags=http-server \
  --metadata=startup-script="$startup_script" || true

gcloud compute firewall-rules create allow-http --allow=tcp:80 --target-tags=http-server --quiet || true

echo "➡ VM créée. Récupérez l'IP :"
gcloud compute instances describe "$GCP_VM_NAME" --zone "$GCP_ZONE" --format='get(networkInterfaces[0].accessConfigs[0].natIP)'
```

### `scripts/01_wp_bootstrap.sh`
```bash
#!/usr/bin/env bash
set -euo pipefail

cd /var/www/html

# attendre DB
until mysqladmin ping -h"${DB_HOST}" -u"${DB_USER}" -p"${DB_PASSWORD}" --silent; do
  echo "⏳ DB..."; sleep 2; done

echo "➡ WP core"
[ -f wp-settings.php ] || wp core download --allow-root

if [ ! -f wp-config.php ]; then
  wp config create --dbname="$DB_NAME" --dbuser="$DB_USER" --dbpass="$DB_PASSWORD" --dbhost="$DB_HOST" --allow-root \
    --extra-php <<PHP
if (!defined('WP_ENVIRONMENT_TYPE')) define('WP_ENVIRONMENT_TYPE', getenv('WP_ENV') ?: 'production');
define('FS_METHOD', 'direct');
PHP
fi

if ! wp core is-installed --allow-root; then
  wp core install --url="$WP_SITE_URL" --title="$WP_SITE_TITLE" \
    --admin_user="$WP_ADMIN_USER" --admin_password="$WP_ADMIN_PASSWORD" --admin_email="$WP_ADMIN_EMAIL" \
    --skip-email --allow-root
fi

echo "➡ Plugins"
wp plugin install advanced-custom-fields --activate --allow-root
wp plugin install brevo --activate --allow-root || wp plugin install mailin --activate --allow-root || true

# Config Brevo si clé présente
if [ -n "${BREVO_API_KEY:-}" ]; then
  wp option update sib_api_key_v3 "$BREVO_API_KEY" --allow-root || true
  wp option patch update sib_home_option from_email "$BREVO_SENDER_EMAIL" --allow-root || true
  wp option patch update sib_home_option from_name "$BREVO_SENDER_NAME" --allow-root || true
fi

wp rewrite structure '/%postname%/' --allow-root
wp option update blog_public 1 --allow-root

# MU healthcheck
mkdir -p wp-content/mu-plugins
if [ ! -f wp-content/mu-plugins/healthcheck.php ]; then
  cat > wp-content/mu-plugins/healthcheck.php <<'PHP'
<?php
add_action('init', function(){
  if (($_SERVER['REQUEST_URI'] ?? '') === '/healthz') {
    header('Content-Type: application/json');
    $ok = (bool) get_option('siteurl');
    http_response_code($ok ? 200 : 500);
    echo json_encode(['status' => $ok ? 'OK' : 'DB ERROR', 'env' => (defined('WP_ENVIRONMENT_TYPE')?WP_ENVIRONMENT_TYPE:'prod')]);
    exit;
  }
});
PHP
fi

echo "✅ bootstrap terminé"
```

### `scripts/02_map_acf_from_template.js`
```js
/* eslint-disable no-console */
const fs = require('fs');
const path = require('path');
const AdmZip = require('adm-zip');
const cheerio = require('cheerio');
const { execSync } = require('child_process');

const THEME_SLUG = process.argv.includes('--theme') ? process.argv[process.argv.indexOf('--theme') + 1] : (process.env.WP_THEME_SLUG || 'auto-theme');
const zipPath = path.resolve('input/template.zip');
const tmp = path.resolve('input/.extract');
const themeDir = path.join('wp-content', 'themes', THEME_SLUG);
const acfJsonDir = path.join(themeDir, 'acf-json');

if (!fs.existsSync(zipPath)) {
  console.error('❌ input/template.zip introuvable');
  process.exit(1);
}

fs.rmSync(tmp, { recursive: true, force: true });
fs.mkdirSync(tmp, { recursive: true });
new AdmZip(zipPath).extractAllTo(tmp, true);

// heuristique : cherche index.html à la racine extraite
const candidates = ['index.html', 'Index.html'];
let htmlFile = null;
for (const c of candidates) {
  const p = path.join(tmp, c);
  if (fs.existsSync(p)) { htmlFile = p; break; }
}
if (!htmlFile) {
  // fallback: 1er .html trouvé
  const files = fs.readdirSync(tmp).filter(f => f.endsWith('.html'));
  if (files[0]) htmlFile = path.join(tmp, files[0]);
}
if (!htmlFile) throw new Error('Aucun fichier HTML trouvé dans le zip');

const $ = cheerio.load(fs.readFileSync(htmlFile, 'utf8'));

// Field group minimal (ex: hero title + images)
const fieldGroup = {
  key: 'group_home_' + Date.now(),
  title: 'Homepage Fields',
  fields: [],
  location: [[{ param: 'page_type', operator: '==', value: 'front_page' }]],
};

const h1 = $('h1').first().text().trim();
if (h1) {
  fieldGroup.fields.push({ key: 'field_hero_title', name: 'hero_title', label: 'Hero Title', type: 'text' });
}
$('img').slice(0, 6).each((i, el) => {
  fieldGroup.fields.push({ key: `field_img_${i}`, name: `image_${i}`, label: `Image ${i}`, type: 'image', return_format: 'url' });
});

fs.mkdirSync(acfJsonDir, { recursive: true });
fs.writeFileSync(path.join(acfJsonDir, 'group-home.json'), JSON.stringify(fieldGroup, null, 2));
console.log('✅ ACF JSON généré →', path.join(acfJsonDir, 'group-home.json'));

// Crée la page d'accueil + renseigne champs simples
function wp(cmd) {
  return execSync(`docker-compose exec -T php wp ${cmd} --allow-root`, { stdio: 'pipe' }).toString().trim();
}
const pageId = wp(`post list --post_type=page --fields=ID,post_title | awk '$$2=="Accueil" {print $$1}'`) || wp(`post create --post_type=page --post_title='Accueil' --post_status=publish --porcelain`);
wp(`option update show_on_front 'page'`);
wp(`option update page_on_front ${pageId}`);
if (h1) {
  wp(`post meta update ${pageId} hero_title "${h1.replace(/"/g,'\\"')}"`);
  wp(`post meta update ${pageId} _hero_title field_hero_title`);
}
console.log('✅ Page Accueil ID=', pageId);
```

### `wp-content/mu-plugins/healthcheck.php`
```php
<?php
add_action('init', function(){
  if (($_SERVER['REQUEST_URI'] ?? '') === '/healthz') {
    header('Content-Type: application/json');
    $ok = (bool) get_option('siteurl');
    http_response_code($ok ? 200 : 500);
    echo json_encode(['status' => $ok ? 'OK' : 'DB ERROR']);
    exit;
  }
});
```

### Thème — `wp-content/themes/auto-theme/style.css`
```css
/*
Theme Name: Auto Theme
Theme URI: https://example.com
Author: Sooatek
Author URI: https://sooatek.com
Description: Thème généré automatiquement depuis un template HTML
Version: 0.1.0
Text Domain: auto-theme
*/

/* Breakpoints */
@media (min-width: 576px) {}
@media (min-width: 768px) {}
@media (min-width: 992px) {}
@media (min-width: 1200px) {}
```

### Thème — `functions.php`
```php
<?php
add_action('after_setup_theme', function(){
  add_theme_support('title-tag');
  add_theme_support('post-thumbnails');
  register_nav_menus(['primary' => __('Primary Menu','auto-theme')]);
});

add_action('wp_enqueue_scripts', function(){
  wp_enqueue_style('auto-style', get_stylesheet_uri(), [], '0.1.0');
});
```

### Thème — `header.php`
```php
<!DOCTYPE html>
<html <?php language_attributes(); ?>>
<head>
  <meta charset="<?php bloginfo('charset'); ?>" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <?php wp_head(); ?>
</head>
<body <?php body_class(); ?>>
<header class="site-header">
  <div class="wrap">
    <h1 class="site-title"><a href="<?php echo esc_url(home_url('/')); ?>"><?php bloginfo('name'); ?></a></h1>
    <?php wp_nav_menu(['theme_location'=>'primary']); ?>
  </div>
</header>
<main class="site-main">
```

### Thème — `footer.php`
```php
</main>
<footer class="site-footer">
  <div class="wrap">© <?php echo date('Y'); ?> <?php bloginfo('name'); ?></div>
</footer>
<?php wp_footer(); ?>
</body>
</html>
```

### Thème — `index.php`
```php
<?php get_header(); ?>
<section class="hero">
  <div class="wrap">
    <h1><?php echo esc_html(get_field('hero_title') ?: get_bloginfo('description')); ?></h1>
  </div>
</section>
<?php if (have_posts()) : while (have_posts()) : the_post(); ?>
<article <?php post_class(); ?>>
  <h2><a href="<?php the_permalink(); ?>"><?php the_title(); ?></a></h2>
  <div class="entry"><?php the_excerpt(); ?></div>
</article>
<?php endwhile; endif; ?>
<?php get_footer(); ?>
```

### Thème — `page.php`
```php
<?php get_header(); ?>
<article <?php post_class(); ?>>
  <h1><?php the_title(); ?></h1>
  <div class="entry"><?php the_content(); ?></div>
</article>
<?php get_footer(); ?>
```

### `tests/playwright/playwright.config.ts`
```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  timeout: 60000,
  use: { baseURL: process.env.BASE_URL || 'http://localhost:8000' },
  projects: [
    { name: 'Desktop', use: devices['Desktop Chrome'] },
    { name: 'Tablet', use: { ...devices['iPad (gen 7)'] } },
    { name: 'Mobile', use: devices['iPhone 13'] },
  ],
});
```

### `tests/playwright/e2e.spec.ts`
```ts
import { test, expect } from '@playwright/test';

test('home responds and shows hero', async ({ page, baseURL }) => {
  const url = baseURL || 'http://localhost:8000';
  const res = await page.goto(url!, { waitUntil: 'domcontentloaded' });
  expect(res?.ok()).toBeTruthy();
  await expect(page.locator('body')).toBeVisible();
});
```

### `.github/workflows/ci.yml`
```yaml
name: ci
on:
  push:
    branches: [ main ]
  pull_request:
  workflow_dispatch:

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    services:
      db:
        image: mariadb:10.11
        env:
          MYSQL_DATABASE: wordpress
          MYSQL_USER: wp
          MYSQL_PASSWORD: wp
          MYSQL_ROOT_PASSWORD: root
      php:
        image: ghcr.io/shivammathur/php:8.3-fpm
      nginx:
        image: nginx:alpine
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20
      - name: Install node deps
        run: npm ci
      - name: Playwright browsers
        run: npx playwright install --with-deps
      - name: Lint
        run: npm run lint
      - name: Tests E2E
        run: npm run test:e2e
```

### `package.json`
```json
{
  "name": "html-to-wp-generator-seed",
  "private": true,
  "scripts": {
    "lint": "echo 'lint placeholder (add eslint/stylelint if needed)'",
    "test:e2e": "playwright test"
  },
  "devDependencies": {
    "@playwright/test": "^1.47.0",
    "adm-zip": "^0.5.12",
    "cheerio": "^1.0.0"
  }
}
```

### `composer.json`
```json
{
  "name": "sooatek/html-to-wp-generator-seed",
  "require-dev": {
    "squizlabs/php_codesniffer": "^3.10"
  },
  "scripts": {
    "lint": [
      "phpcs --standard=WordPress wp-content/themes/auto-theme"
    ]
  }
}
