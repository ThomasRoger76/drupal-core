---
name: drupal-core
description: Use when building custom Drupal modules, structuring .info.yml/.permissions.yml/.services.yml, implementing routes/controllers with dependency injection, choosing between hooks and EventSubscribers, using the Plugin system, Forms API with AJAX and #states, Entity API, Database API, Config vs State API, Cache tags/contexts, or creating custom services in Drupal 8-11+
---

# Drupal Core — Architecture & Référence Complète

## Overview

Référentiel complet du développement backend Drupal 8-11+ : structure de module, DI via constructeur, plugin system, hooks vs events, Forms API, Database API, Entity API, Config/State, Cache. Couvre l'implémentation et les **évolutions entre versions majeures**.

## Quick Decision Table

| Besoin | Outil Drupal | Référence |
|--------|-------------|-----------|
| Créer un module from scratch | Structure + `.info.yml` | [module-structure.md](module-structure.md) |
| URL → page / JSON response | Route + Controller | [routing-controllers.md](routing-controllers.md) |
| Route vers un formulaire (sans Controller) | `_form:` dans routing.yml | [module-structure.md](module-structure.md) |
| Créer une permission custom | `.permissions.yml` | [module-structure.md](module-structure.md) |
| Ajouter un lien de menu admin | `.links.menu.yml` | [module-structure.md](module-structure.md) |
| Bloc / widget / formatter réutilisable | Plugin (Block, FieldFormatter…) | [plugin-system.md](plugin-system.md) |
| Modifier un formulaire existant | `hook_form_alter` / `hook_form_FORM_ID_alter` | [hooks-events.md](hooks-events.md) |
| Réagir à la sauvegarde d'une entité | `hook_entity_presave` | [hooks-events.md](hooks-events.md) |
| Réagir à une requête/réponse HTTP | `EventSubscriber` (KernelEvents) | [hooks-events.md](hooks-events.md) |
| Workflow / State Machine transition | `EventSubscriber` | [hooks-events.md](hooks-events.md) |
| Ajouter CSS/JS à toutes les pages | `hook_page_attachments` | [hooks-events.md](hooks-events.md) |
| Formulaire custom standalone | `FormBase` / `ConfigFormBase` | [forms-api.md](forms-api.md) |
| Afficher/cacher éléments de form en JS | `#states` API | [forms-api.md](forms-api.md) |
| Charger / requêter des entités Drupal | `EntityTypeManager` + `EntityQuery` | [services-internal-api.md](services-internal-api.md) |
| Requête SQL sur table custom / JOIN / agrégation | Database API | [database-api.md](database-api.md) |
| Créer une table custom en base | `hook_schema` + `hook_update_N` | [database-api.md](database-api.md) |
| Créer un service réutilisable | `.services.yml` + classe PHP | [services-internal-api.md](services-internal-api.md) |
| Settings admin exportables en YAML | Config API | [services-internal-api.md](services-internal-api.md) |
| Donnée runtime volatile (timestamp…) | State API | [services-internal-api.md](services-internal-api.md) |
| Performance — ne pas servir du périmé | Cache tags + contexts + max-age | [services-internal-api.md](services-internal-api.md) |

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Raison |
|---------------------|------------------|--------|
| `\Drupal::service()` dans une classe | Injection via `__construct()` | Testabilité, découplage |
| `\Drupal::entityTypeManager()` dans un Controller | Injecter `EntityTypeManagerInterface` | Idem |
| `drupal_set_message()` | `\Drupal::messenger()->addStatus()` | Supprimé Drupal 9 |
| `db_query()` | `EntityQuery` ou service `database` | Supprimé Drupal 8 |
| Hooks en classe PHP sans `#[Hook]` (D8-D10) | `.module` (D8-D10) ou classe `#[Hook]` (D11+) | Sans l'attribute, Drupal ne découvre pas les hooks en classe |
| `@Block(...)` annotation sur PHP 8.2+ / D11+ | `#[Block(...)]` attribute PHP | Annotations dépréciées D11 |
| Render array sans `#cache` sur contenu dynamique | Toujours définir tags + contexts | Contenu périmé en production |
| `hook_node_presave` si générique | `hook_entity_presave` | Plus générique, même résultat |

## Évolution par Version Majeure

| Feature | D8 | D9 | D10 | D11 |
|---------|----|----|-----|-----|
| PHP minimum | 7.0 | 7.3 | 8.1 | 8.3 |
| Plugin annotations | ✅ seul | ✅ | ✅ | ⚠️ déprécié |
| PHP Attributes (#[...]) | ❌ | ❌ | ✅ optionnel | ✅ **standard** |
| Symfony version | 3.x | 4.x | 6.x | 7.x |
| `drupal_set_message()` | ✅ | ❌ supprimé | ❌ | ❌ |
| `db_query()` | ❌ supprimé | ❌ | ❌ | ❌ |
| Constructor promotion PHP | ❌ | ❌ | ✅ | ✅ |
| Hooks via `#[Hook]` attribute | ❌ | ❌ | ❌ | ✅ **disponible** |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Bugs trouvés en usage réel + prévention. Ajouter une entrée après chaque correction issue d'un projet.
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions du skill (v3.1 courante).

**Workflow :** bug trouvé en projet → corriger le fichier source → ajouter entrée dans `lessons.md` → incrémenter CHANGELOG.

## See Also

- `drupal-theming` — Twig, preprocess, libraries
- `drupal-config` — Config Management System, UUID conflicts, overrides
- `drupal-testing` — PHPUnit, Kernel tests, Functional tests
- `drupal-security` — CSRF, XSS, permissions, accès API
- `drupal-tooling` — DDEV, Drush, déploiement
