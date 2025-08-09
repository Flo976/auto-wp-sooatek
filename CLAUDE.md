# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an automated WordPress deployment project that converts static HTML templates to responsive WordPress themes. The system uses Docker Compose with containerized WordPress stack (Nginx, PHP-FPM 8.3, MariaDB) for local development and automated deployment to Google Cloud Platform.

## Essential Commands

Based on the project's Makefile workflow:

### Local Development
- `make up` - Build and start Docker containers (Nginx on http://localhost:8000)
- `make install-wp` - Run WordPress installation + plugin configuration via WP-CLI
- `make apply-theme` - Parse HTML template, generate ACF fields, and activate theme
- `make down` - Stop containers
- `make reset` - Complete environment reset (stops containers + removes DB volume)

### Testing & Quality Assurance
- `make test` - Run linters (PHPCS, ESLint, Stylelint) + Playwright end-to-end tests
- Tests verify responsive layouts on mobile/tablet/desktop viewports
- Playwright config: `tests/playwright/e2e.spec.ts` and `playwright.config.ts`

### Deployment
- `make deploy-dev` - Deploy to Google Cloud VM (requires GCP configuration in `deploy/dev.sh`)

## Architecture & Core Components

### Docker Stack
- **Nginx**: Web server (port 8000 locally, 80 in production)
- **PHP-FPM 8.3**: WordPress runtime with WP-CLI pre-installed
- **MariaDB 10.11**: Database with persistent volume storage
- **Optional Mailpit**: Email capture for local development

### WordPress Integration
- **Advanced Custom Fields (ACF)**: Field definitions stored as JSON in theme's `acf-json/` directory
- **Brevo Plugin**: Email service integration via API key configuration
- **Health Endpoint**: `/healthz` endpoint for monitoring (returns DB status + WP version)

### Template Processing
- `scripts/02_map_acf_from_template.js` - Node.js script that parses HTML templates
- Automatically generates ACF field groups from HTML structure
- Imports images to WordPress media library
- Creates homepage with pre-filled content from template

### Theme Development
- Custom WordPress theme located in `wp-content/themes/<slug>/`
- Responsive CSS with breakpoints: mobile (<576px), tablet (~768px), desktop (>992px)
- ACF fields integrated via `the_field()` functions
- Theme activation triggers ACF JSON loading

### Configuration Management
- Environment variables in `.env` (copy from `.env.example`)
- Required variables: DB credentials, WP admin user, Brevo API key, site URL
- WP-CLI automation handles all WordPress installation and configuration

### CI/CD Pipeline
- GitHub Actions workflow in `.github/workflows/ci.yml`
- Automated linting, testing, and deployment on main branch
- GCP deployment requires service account configuration in GitHub Secrets

## Development Workflow

1. **Initial Setup**: `make init` → configure `.env` → `make up`
2. **WordPress Installation**: `make install-wp` (one-time setup)
3. **Theme Development**: Edit files in `wp-content/themes/<slug>/`, then `make apply-theme`
4. **Testing**: `make test` to run quality checks and responsive tests
5. **Deployment**: `make deploy-dev` for GCP deployment

## Important Notes

- WordPress core files are downloaded dynamically via WP-CLI (not in repository)
- ACF field definitions are version-controlled as JSON files
- Brevo API key is injected automatically during WordPress installation
- Health monitoring available at `/healthz` endpoint
- All automation scripts use WP-CLI for consistency and reliability
- Template parsing is idempotent - can be re-run safely

## Troubleshooting

- **DB Connection Issues**: Ensure containers are running (`make up`) and DB_HOST=db in environment
- **Port Conflicts**: Default port 8000 can be changed in docker-compose.yml
- **ACF Changes**: Field modifications are auto-saved to acf-json/ directory
- **File Permissions**: WordPress uploads belong to www-data user in containers
- **Email Testing**: Brevo configuration can be tested via WordPress admin password reset