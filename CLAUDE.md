# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a seed repository for an automated HTML-to-WordPress generator project. The mission is to create a reusable tool that transforms any static HTML/CSS template into a responsive WordPress theme with ACF (free), Brevo configuration, automated tests, and GCP deployment.

The repository contains the complete infrastructure template including Docker stack (Nginx, PHP-FPM 8.3, MariaDB), WP-CLI automation scripts, theme boilerplate, Playwright tests, and CI/CD pipeline. This serves as the foundation for generating WordPress projects automatically.

## Essential Commands

Based on the project's Makefile workflow:

### Local Development
- `make init` - Copy `.env.example` to `.env` (first-time setup)
- `make up` - Build and start Docker containers (Nginx on http://localhost:8000)
- `make install-wp` - Run WordPress installation + plugin configuration via WP-CLI
- `make apply-theme` - Parse HTML template, generate ACF fields, and activate theme
- `make down` - Stop containers
- `make reset` - Complete environment reset (stops containers + removes DB volume)

### Testing & Quality Assurance
- `make test` - Run linters + Playwright end-to-end tests across mobile/tablet/desktop viewports
- `make node-ci` - Install Node.js dependencies in container
- Individual test debugging: `npx playwright test --headed --project=Desktop`

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

### Template Processing Pipeline
- **Input**: HTML template placed in `input/template.zip`
- `scripts/02_map_acf_from_template.js` - Core parsing engine using Cheerio
- Automatically detects repeating patterns (team members, features) → ACF Repeater fields
- Maps images to ACF Image fields, text content to ACF Text fields
- Generates ACF JSON schema in `wp-content/themes/auto-theme/acf-json/`
- Creates WordPress homepage and pre-fills ACF fields with extracted content

### Theme Architecture
- **Base Theme**: `wp-content/themes/auto-theme/` (customizable via WP_THEME_SLUG)
- **Responsive Strategy**: Mobile-first with breakpoints at 576px, 768px, 992px, 1200px
- **ACF Integration**: Uses `get_field()` and `the_field()` for dynamic content
- **JSON Schema**: ACF field definitions version-controlled as JSON files

### Configuration Management
- Environment variables in `.env` (copy from `.env.example`)
- Required variables: DB credentials, WP admin user, Brevo API key, site URL
- WP-CLI automation handles all WordPress installation and configuration

### CI/CD Pipeline
- GitHub Actions workflow in `.github/workflows/ci.yml`
- Automated linting, testing, and deployment on main branch
- GCP deployment requires service account configuration in GitHub Secrets

## Development Workflow

### First-Time Setup
1. `make init` - Initialize environment variables
2. Configure `.env` with appropriate values (DB credentials, admin user, etc.)
3. `make up` - Start Docker containers
4. `make install-wp` - Install WordPress core + plugins + Brevo configuration

### Template Processing Workflow
1. Place HTML template as `input/template.zip`
2. `make apply-theme` - Parse template → generate ACF fields → activate theme
3. Verify results at http://localhost:8000
4. Manually adjust theme files in `wp-content/themes/auto-theme/` if needed

### Quality Assurance
1. `make test` - Run complete test suite (linting + E2E tests)
2. Review Playwright test results across all viewports
3. Check `/healthz` endpoint returns HTTP 200

### Generator Development Context
This seed repository is designed to be replicated by a future HTML-to-WordPress generator tool. The structure serves as a template for automated project generation with customizable theme names, configuration values, and GCP deployment settings.

## Important Notes

- **WordPress Core**: Downloaded dynamically via WP-CLI (not version-controlled)
- **ACF Schema**: Field definitions stored as JSON in `acf-json/` for version control and performance
- **Template Requirements**: HTML templates must be provided in `input/template.zip`
- **Node Service**: Uses Microsoft Playwright Docker image for consistent testing environment
- **Brevo Integration**: API key auto-injected via WP-CLI options during installation
- **Idempotent Operations**: Template parsing and theme application can be re-run safely
- **Health Monitoring**: `/healthz` endpoint returns DB status and environment info

## Troubleshooting

### Common Issues
- **DB Connection Errors**: Run `make down && make up` to restart containers cleanly
- **Port 8000 Conflicts**: Modify port mapping in `docker-compose.yml` if needed
- **Missing Template**: Ensure `input/template.zip` exists before running `make apply-theme`
- **File Permissions**: WordPress uploads owned by www-data; use `sudo chown -R $USER:$USER wp-content` if needed

### Development Debugging
- **Container Logs**: `docker-compose logs php` or `docker-compose logs nginx`
- **WP-CLI Access**: `docker-compose exec php wp --info --allow-root`
- **Playwright Debugging**: Add `--headed` flag to see browser interactions
- **ACF Sync Issues**: Use WP Admin → ACF → Sync to reload JSON field definitions

### Quality Assurance Checklist
- HTTP 200 response from `/healthz` endpoint
- Responsive layouts validated on mobile/tablet/desktop (Playwright tests pass)
- ACF fields properly mapped and pre-filled on homepage
- No PHPCS/ESLint/Stylelint errors in theme code