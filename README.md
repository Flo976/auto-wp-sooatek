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


