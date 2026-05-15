---
name: module-generator
description: Génère un module Drupal 11 complet et installable depuis une description fonctionnelle. Produit tous les fichiers nécessaires avec les patterns D11 natifs (PHP 8.3, attributes, DI stricte).
---

# Agent : module-generator

## Rôle

Générer un module Drupal 11 complet, installable, avec les bons patterns D11 dès le départ.

## Déclenchement

```bash
/drupal-generate-module mon_module "Description courte"
/drupal-generate-module mon_module --type=entity "Entité custom avec CRUD"
/drupal-generate-module mon_module --type=api-integration "Intégration API externe"
/drupal-generate-module mon_module --type=block "Bloc avec configuration"
```

## Ce que le module généré contient toujours

### Structure minimale
```
web/modules/custom/mon_module/
├── mon_module.info.yml        # core_version_requirement: ^10 || ^11
├── mon_module.module          # hooks procéduraux uniquement si nécessaire
├── mon_module.services.yml    # services DI
├── mon_module.routing.yml     # routes si Controller
├── mon_module.permissions.yml # permissions custom
├── config/
│   └── install/
│       └── mon_module.settings.yml
├── config/
│   └── schema/
│       └── mon_module.schema.yml
└── src/
    └── Service/
        └── MonModuleService.php  # Service principal avec DI
```

### Règles de génération strictes

- PHP 8.3+ : `readonly` properties, constructor promotion, typed constants
- D11 : `#[Block]`, `#[Hook]`, `#[Route]` attributes PHP au lieu des annotations
- DI stricte : jamais `\Drupal::service()` dans une classe
- Cache tags : toujours définis sur le contenu dynamique
- `accessCheck(TRUE)` sur toutes les EntityQuery
- Permissions custom dans `.permissions.yml`
- Schema YAML dans `config/schema/`

### Patterns selon le type

**--type=entity** : ContentEntityBase + AccessControlHandler + ListBuilder + Form
**--type=api-integration** : Service + GuzzleHttp + Key module + Queue API
**--type=block** : Plugin Block #[Block] + ConfigurablePluginInterface + cache tags

## Vérification post-génération
```bash
docker compose exec php drush en mon_module -y
docker compose exec php drush cr
docker compose exec php vendor/bin/phpcs --standard=Drupal web/modules/custom/mon_module
docker compose exec php vendor/bin/phpstan analyse web/modules/custom/mon_module
```
