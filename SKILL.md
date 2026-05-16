---
name: drupal-core
description: Use when building custom Drupal modules, structuring .info.yml/.permissions.yml/.services.yml, implementing routes/controllers with dependency injection, choosing between hooks and EventSubscribers, using the Plugin system (Block, FieldType, FieldFormatter, QueueWorker), Forms API with AJAX and #states, Entity API, Database API, Config vs State API, Cache tags/contexts, creating custom services, sending emails with Symfony Mailer, managing editorial workflows with Content Moderation, configuring Layout Builder programmatically, handling multilingual entities and translations, or writing custom Drush commands in Drupal 8-11+
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
| Bloc / widget réutilisable | Plugin `Block` avec `#[Block]` | [plugin-system.md](plugin-system.md) |
| Formatter de champ custom (affichage) | Plugin `FieldFormatter` + `#[FieldFormatter]` | [plugin-system.md](plugin-system.md) |
| Type de champ custom (stockage + widget + formatter) | Plugin `FieldType` + `FieldWidget` + `FieldFormatter` | [plugin-system.md](plugin-system.md) |
| Données temporaires post-migration (schéma libre) | `hook_post_update_NAME` dans `.post_update.php` | [module-structure.md](module-structure.md) |
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
| Modifier le schéma d'un autre module | `hook_schema_alter()` | [database-api.md](database-api.md) |
| Créer un service réutilisable | `.services.yml` + classe PHP | [services-internal-api.md](services-internal-api.md) |
| Settings admin exportables en YAML | Config API | [services-internal-api.md](services-internal-api.md) |
| Donnée runtime volatile (timestamp…) | State API | [services-internal-api.md](services-internal-api.md) |
| Performance — ne pas servir du périmé | Cache tags + contexts + max-age | [services-internal-api.md](services-internal-api.md) |
| Traitement lourd sans bloquer le navigateur | Queue API (`QueueInterface`, `QueueWorker` plugin) | [hooks-events.md](hooks-events.md) |
| Créer une queue nommée programmatiquement | `queue_factory` service + `$queueFactory->get('nom_queue')` | [hooks-events.md](hooks-events.md) |
| Traitement par lots avec barre de progression | Batch API (`batch_set()`, `operations`, `finished`) | [hooks-events.md](hooks-events.md) |
| Commande Drush custom | `Commands` class + `drush.services.yml` | [routing-controllers.md](routing-controllers.md) |
| Endpoint REST/JSON pour entité Drupal | JSON:API (module core) | [json-api.md](json-api.md) |
| Filtrer/inclure via JSON:API | `?filter[field]=val&include=ref` | [json-api.md](json-api.md) |
| CORS pour frontend découplé | `cors.config:` dans services.yml | [json-api.md](json-api.md) |
| Créer/charger un Media programmatiquement | `Media::create()`, `file_url_generator` | [media-api.md](media-api.md) |
| Envoyer un email D11 (Symfony Mailer) | `EmailFactory`, `EmailBuilderBase` | [symfony-mailer.md](symfony-mailer.md) |
| Template Twig pour un email | `@symfony_mailer/base.html.twig` | [symfony-mailer.md](symfony-mailer.md) |
| Gérer les états éditoriaux (draft/published/archived) | Content Moderation module core | [content-moderation.md](content-moderation.md) |
| Réagir à une transition de modération | `ContentModerationStateChangedEvent` | [content-moderation.md](content-moderation.md) |
| Changer l'état d'un nœud programmatiquement | `$node->set('moderation_state', 'published')` | [content-moderation.md](content-moderation.md) |
| Value Object immuable (DTO sans setter) | `readonly class` PHP 8.3 | [services-internal-api.md](services-internal-api.md) |
| Constantes typées PHP 8.3 | `const string STATE = 'draft'` | [services-internal-api.md](services-internal-api.md) |
| Activer Layout Builder sur un type de contenu | `$display->enableLayoutBuilder()->setOverridable()` | [layout-builder.md](layout-builder.md) |
| Ajouter un bloc à un layout programmatiquement | `Section` + `SectionComponent` | [layout-builder.md](layout-builder.md) |
| Vérifier si un nœud a un layout personnalisé | `LayoutEntityHelperTrait::getSectionStorageForEntity()` | [layout-builder.md](layout-builder.md) |
| Désactiver Layout Builder via code | `$display->disableLayoutBuilder()->save()` | [layout-builder.md](layout-builder.md) |
| Réagir au rendu d'un composant Layout Builder | `EventSubscriber` sur `LayoutBuilderEvents::SECTION_COMPONENT_BUILD_RENDER_ARRAY` | [layout-builder.md](layout-builder.md) |
| Charger un nœud dans la langue courante | `$node->getTranslation($langcode)` | [multilingual.md](multilingual.md) |
| EntityQuery filtré par langue | `->condition('langcode', $langcode)` | [multilingual.md](multilingual.md) |
| Créer une traduction programmatiquement | `$node->addTranslation('fr', [...])` | [multilingual.md](multilingual.md) |
| Override de config par langue | `LanguageManager::getLanguageConfigOverride()` | [multilingual.md](multilingual.md) |
| Générer une URL dans une langue spécifique | `Url::fromRoute(..., ['language' => $langue])` | [multilingual.md](multilingual.md) |
| Traduire une chaîne dans un service PHP | `$this->t()` via `StringTranslationTrait` | [multilingual.md](multilingual.md) |

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
- `drupal-tooling` — Drush CLI (updb, cim, cr, deploy, custom commands), DDEV
- `drupal-migration` — Migrate API (importer des données via plugins)
- `drupal-security` — CSRF, XSS, permissions, accès API
- `drupal-docker` — Environnement Docker Compose local
- `drupal-views` — Exposer des données à Views, `hook_views_data()`
- `drupal-performance` — Cache tags/contexts dans les render arrays, BigPipe
- `drupal-content-modeling` — Custom Entities, Paragraphs, Layout Builder
- `drupal-multilingual` — `getTranslation()`, `StringTranslationTrait`
- `drupal-api` — JSON:API, REST Resources, permissions headless
- `drupal-search` — `hook_search_api_query_alter()`, custom processors
