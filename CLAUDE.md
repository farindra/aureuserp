# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**AureusERP** — OB ERP (Ocean Bearings) instance. An open-source ERP built on:
- **Laravel 13** (streamlined Laravel 11+ structure — no Kernel.php, middleware registered in `bootstrap/app.php`)
- **FilamentPHP 5** (admin panel, resources, pages, widgets, clusters)
- **Livewire 4** (dynamic UI components)
- **Pest 4** (testing framework)
- **TailwindCSS 4**
- PHP 8.3+, SQLite (development), MySQL 8+ (production)

## Common Commands

```bash
# Start all services concurrently (server + queue + logs + vite)
composer run dev

# Individual services
php artisan serve
npm run dev
npm run build
php artisan queue:listen --tries=1
php artisan pail --timeout=0

# Code formatting (MUST run before finalizing changes)
vendor/bin/pint --dirty --format agent

# Run all tests
php artisan test --compact

# Run a single test or filter by name
php artisan test --compact --filter=testName

# Run a specific plugin's test suite
php artisan test --testsuite=AccountFeature

# Create tests
php artisan make:test --pest {name}          # feature test
php artisan make:test --pest --unit {name}   # unit test
```

## Plugin System Architecture

All business logic lives in `plugins/webkul/` — the core `app/` directory only contains middleware, base providers, and the two Filament panel providers.

Each plugin follows this structure:
```
plugins/webkul/{plugin-name}/
  composer.json           # declares Webkul\{PluginName}\ namespace
  src/
    {Name}ServiceProvider.php   # extends PackageServiceProvider, configures package
    {Name}Plugin.php            # implements Filament Plugin interface
    Filament/
      Clusters/           # navigation clusters grouping related resources/pages
      Resources/          # Filament CRUD resources
      Pages/
      Widgets/
    Models/
    Enums/
    Policies/
    Livewire/
    Settings/
  database/
    migrations/
    seeders/
    factories/
    settings/
  resources/
    lang/
    views/
  config/
  routes/
```

### How Plugins Register with Filament

1. `{Name}ServiceProvider` extends `PackageServiceProvider` (from `plugin-manager`) and calls `configureCustomPackage()` to declare dependencies, migrations, views, routes, and install/uninstall commands.
2. `{Name}Plugin` implements `Filament\Contracts\Plugin` and conditionally registers Filament resources/pages/clusters/widgets only when the plugin is installed (`Package::isPluginInstalled($this->getId())`).
3. Registration happens automatically via `Panel::configureUsing()` in the service provider's `packageRegistered()` method.
4. The root `composer.json` uses `wikimedia/composer-merge-plugin` to merge all `plugins/*/*/composer.json` files.

### Plugin Installation

```bash
php artisan {plugin-name}:install      # installs with dependency resolution
php artisan {plugin-name}:uninstall    # removes tables and data
php artisan erp:install                # full system installation
```

### Filament Panels

- **Admin panel** (`/admin`): `app/Providers/Filament/AdminPanelProvider.php` — the main ERP interface
- **Customer panel**: `app/Providers/Filament/CustomerPanelProvider.php`

Navigation groups, global search, RBAC (via `filament-shield`), and MFA are configured in the admin panel provider.

### Plugin Dependencies

Plugins declare dependencies in `configureCustomPackage()` with `->hasDependencies(['other-plugin'])`. During install, dependent plugins are checked/installed first.

## Testing Conventions

Tests live in each plugin's `tests/Feature/` directory. Test suites are declared in `phpunit.xml` (e.g., `AccountFeature`, `InventoryFeature`). Tests use:
- Pest 4 with `beforeEach`/`afterEach` hooks
- Shared helpers in `plugins/webkul/support/tests/Helpers/` (require'd manually)
- `TestBootstrapHelper::ensurePluginInstalled()` to set up plugin state
- `SecurityHelper` to authenticate users with specific permissions for API tests
- Route naming pattern: `admin.api.v1.{plugin}.{resource}.{action}`

## Code Style

- Laravel Pint with `laravel` preset; `=>` operators align, no space in string concatenation
- PHP 8 constructor property promotion, explicit return types on all methods
- No `env()` outside config files — use `config('key')` everywhere
- Enums use TitleCase keys
- `Model::query()` over `DB:::`; eager load to avoid N+1 problems
- Form Request classes for all validation (never inline in controllers)
- Use `php artisan make:` commands to create new files, always pass `--no-interaction`

## Key Packages

| Package | Purpose |
|---|---|
| `bezhansalleh/filament-shield` | RBAC / role-based permissions |
| `filament/spatie-laravel-settings-plugin` | Settings management |
| `spatie/laravel-query-builder` | API query building |
| `maatwebsite/excel` | Import/export |
| `barryvdh/laravel-dompdf` | PDF generation |
| `milon/barcode` | Barcode generation |
| `knuckleswtf/scribe` | API documentation |
| `spatie/eloquent-sortable` | Sortable models |
