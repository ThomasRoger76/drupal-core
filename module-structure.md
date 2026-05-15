# Structure d'un Module Custom

## Arborescence Complète

```
mon_module/
├── mon_module.info.yml           # Métadonnées (obligatoire)
├── mon_module.module             # Hooks PHP
├── mon_module.install            # hook_schema, hook_install, hook_update_N
├── mon_module.routing.yml        # Routes
├── mon_module.services.yml       # Services custom + event subscribers
├── mon_module.permissions.yml    # Permissions custom
├── mon_module.links.menu.yml     # Liens de menu admin
├── mon_module.links.task.yml     # Onglets (tabs) contextuels
├── mon_module.links.action.yml   # Boutons d'action contextuels
├── mon_module.libraries.yml      # Bibliothèques CSS/JS
├── config/
│   ├── install/                  # Config importée à l'installation
│   │   └── mon_module.settings.yml
│   ├── optional/                 # Config importée si dépendances OK
│   └── schema/
│       └── mon_module.schema.yml # Schéma de validation config
├── src/
│   ├── Controller/
│   ├── Form/
│   ├── Hook/                     # Hooks D11 via #[Hook] attribute (src/Hook/)
│   ├── Plugin/
│   │   ├── Block/
│   │   ├── Field/
│   │   │   ├── FieldFormatter/
│   │   │   └── FieldWidget/
│   │   └── QueueWorker/
│   ├── Entity/
│   ├── EventSubscriber/
│   ├── Service/
│   └── Access/
├── templates/                    # Fichiers .twig
└── tests/
    └── src/
        ├── Unit/
        ├── Kernel/
        └── Functional/
```

---

## `.info.yml` — Anatomie Complète

```yaml
# mon_module.info.yml
name: Mon Module
description: 'Description courte de ce que fait le module.'
type: module
core_version_requirement: ^10 || ^11    # Versions Drupal supportées
package: Custom                         # Groupe dans admin/modules

# Dépendances
dependencies:
  - drupal:node                         # Module Drupal core
  - drupal:views                        # Views core
  - paragraphs:paragraphs               # Module contrib (vendor:machine_name)

# PHP minimum (optionnel) — version string exacte, PAS de syntaxe Composer (^, ~, etc.)
# Drupal utilise version_compare(), pas le parser Composer
php: '8.2'
```

---

## `.permissions.yml` — Permissions Custom

```yaml
# mon_module.permissions.yml
administer mon module:
  title: 'Administrer Mon Module'
  description: 'Configurer les paramètres de Mon Module.'
  restrict access: true           # Masqué aux non-admins dans l'UI

access mon module content:
  title: 'Accéder au contenu Mon Module'
  description: 'Voir le contenu fourni par Mon Module.'

# Permissions dynamiques (callback PHP)
permission_callbacks:
  - Drupal\mon_module\MonModulePermissions::permissions
```

```php
<?php
// src/MonModulePermissions.php
namespace Drupal\mon_module;

use Drupal\Core\StringTranslation\StringTranslationTrait;

final class MonModulePermissions {
  use StringTranslationTrait;

  public function permissions(): array {
    $permissions = [];
    // Permissions dynamiques (ex: une par type de contenu)
    foreach (['article', 'page'] as $bundle) {
      $permissions["edit $bundle content in mon module"] = [
        'title' => $this->t('Modifier le contenu @type dans Mon Module', ['@type' => $bundle]),
      ];
    }
    return $permissions;
  }
}
```

---

## Liens de Navigation

### `.links.menu.yml` — Menu Admin

```yaml
# mon_module.links.menu.yml
mon_module.admin:
  title: 'Mon Module'
  description: 'Paramètres de Mon Module.'
  route_name: mon_module.settings_form
  parent: system.admin_config          # Sous "Configuration" dans /admin/config
  weight: 10

mon_module.admin.settings:
  title: 'Paramètres'
  route_name: mon_module.settings_form
  parent: mon_module.admin
```

### `.links.task.yml` — Onglets (Tabs)

```yaml
# mon_module.links.task.yml — onglets sur les pages existantes
mon_module.settings_tab:
  title: 'Paramètres'
  route_name: mon_module.settings_form
  base_route: mon_module.admin
  weight: 0
```

### `.links.action.yml` — Boutons d'Action

```yaml
# mon_module.links.action.yml — boutons "+ Ajouter X"
mon_module.add_item:
  title: 'Ajouter un item'
  route_name: mon_module.item_add
  appears_on:
    - mon_module.items_list
```

---

## `config/schema/mon_module.schema.yml`

Le schéma rend la config typée, validable et traduisible.

```yaml
# config/schema/mon_module.schema.yml
mon_module.settings:
  type: config_object
  label: 'Paramètres Mon Module'
  mapping:
    max_items:
      type: integer
      label: 'Nombre maximum d''items'
    mode:
      type: string
      label: 'Mode d''affichage'
    enabled:
      type: boolean
      label: 'Activer la fonctionnalité'
    api_endpoint:
      type: string
      label: 'URL de l''API externe'
```

---

## Workflow de Développement

### Créer un module from scratch

```bash
# 1. Créer la structure minimale
mkdir -p web/modules/custom/mon_module/src/{Controller,Form,Plugin/Block}
touch web/modules/custom/mon_module/mon_module.info.yml
touch web/modules/custom/mon_module/mon_module.module

# 2. Activer le module
docker compose exec php drush en mon_module -y
docker compose exec php drush cr

# 3. Développer (cycle standard)
# → coder → docker compose exec php drush cr → tester → recommencer

# 4. Vérifier les logs
docker compose exec php drush watchdog:tail --severity=error
```

### Route pour un formulaire (shortcut `_form`)

```yaml
# Utiliser _form directement sans créer un Controller
mon_module.settings_form:
  path: '/admin/config/mon-module/settings'
  defaults:
    _form: '\Drupal\mon_module\Form\SettingsForm'
    _title: 'Paramètres'
  requirements:
    _permission: 'administer mon module'
  options:
    _admin_route: TRUE
```

---

## Troubleshooting

| Problème | Cause probable | Solution |
|----------|---------------|---------|
| Module n'apparaît pas dans admin/modules | Syntaxe YAML invalide dans `.info.yml` | `drush cr` puis vérifier YAML (indentation) |
| Hook ne se déclenche pas | Cache pas vidé après ajout du hook | `drush cr` (hooks sont mis en cache) |
| Class not found / autoload error | Namespace PSR-4 ne correspond pas au chemin | Vérifier : `Drupal\mon_module\Controller\` → `src/Controller/` |
| Service not found | `.services.yml` mal formaté ou service mal déclaré | `drush cr` + vérifier YAML |
| Config non importée | Config déjà présente en DB (import ignoré) | `drush cim` ou supprimer la config existante |
| Route 404 | Route non enregistrée ou cache routing | `drush cr` (le routing est mis en cache) |
| Hook ordre incorrect | Poids du module pas assez élevé | `hook_module_implements_alter()` ou ajuster le poids |

```bash
# Commandes utiles en dev
docker compose exec php drush cr                                          # Vider tous les caches
docker compose exec php drush php:eval "var_dump(\Drupal::service('mon_module.mon_service'));"
docker compose exec php drush route:list | grep mon_module                # Lister les routes du module
docker compose exec php drush container:debug | grep event_subscriber     # Lister les event subscribers
docker compose exec php drush watchdog:show --type=mon_module             # Logs du module
```

---

## `hook_post_update_NAME` — Plus Flexible que `hook_update_N`

| Hook | Fichier | Quand | Ordre d'exécution |
|------|---------|-------|-------------------|
| `hook_schema()` | `.install` | Tables DB personnalisées | Avant tout |
| `hook_update_N()` | `.install` | Migrations schéma numérotées | `drush updb`, avant `cim` |
| `hook_deploy_N()` | `.deploy.php` | Data/config après import config | `drush deploy`, après `cim` |
| `hook_post_update_NAME()` | `.post_update.php` | Data complexe, nommage libre | Après tous `hook_update_N` |

```php
// mon_module.post_update.php

// Avantage : nommage libre → pas de conflits entre branches git
function mon_module_post_update_migrate_categories(?array &$sandbox): string {
  if (!isset($sandbox['total'])) {
    $sandbox['nids'] = \Drupal::entityQuery('node')
      ->condition('type', 'article')
      ->accessCheck(FALSE)
      ->execute();
    $sandbox['total'] = count($sandbox['nids']);
    $sandbox['processed'] = 0;
  }
  $batch = array_splice($sandbox['nids'], 0, 50);
  foreach (Node::loadMultiple($batch) as $node) {
    $node->set('field_tags', $node->get('field_category')->getValue());
    $node->save();
    $sandbox['processed']++;
  }
  $sandbox['#finished'] = $sandbox['total'] ? $sandbox['processed'] / $sandbox['total'] : 1;
  return "Migré {$sandbox['processed']}/{$sandbox['total']} nœuds.";
}

function mon_module_post_update_clean_orphan_data(): void {
  \Drupal::database()->delete('mon_module_legacy_table')->execute();
}
```
