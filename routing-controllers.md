# Routing & Controllers

## Anatomie d'un `*.routing.yml`

```yaml
# mon_module.routing.yml
mon_module.liste:
  path: '/mon-module/liste'
  defaults:
    _controller: '\Drupal\mon_module\Controller\ListeController::build'
    _title: 'Ma liste'
  requirements:
    _permission: 'access content'

mon_module.admin:
  path: '/admin/config/mon-module'
  defaults:
    _controller: '\Drupal\mon_module\Controller\AdminController::index'
    _title: 'Configuration'
  requirements:
    _custom_access: '\Drupal\mon_module\Access\MonAccess::checkAccess'
  options:
    _admin_route: TRUE
```

### `_permission` vs `_custom_access` vs `_entity_access`

| | `_permission` | `_custom_access` | `_entity_access` |
|--|--------------|-----------------|-----------------|
| Usage | Permission string simple | Logique conditionnelle | Accès basé sur l'entité |
| Exemple | `'administer nodes'` | Vérifie plusieurs conditions | `'node.update'` |
| Méthode | Chaîne dans le YAML | Méthode → `AccessResult` | Opération sur l'entité |

```yaml
# _entity_access — protéger une route selon les droits sur l'entité (plus sûr)
mon_module.node_edit:
  path: '/mon-module/{node}/edit'
  defaults:
    _controller: '\Drupal\mon_module\Controller\EditController::build'
  requirements:
    _entity_access: 'node.update'   # Vérifie que l'user peut modifier CE nœud
  options:
    parameters:
      node:
        type: entity:node
```

---

## Controller — Pattern Standard

```php
<?php
// src/Controller/ListeController.php
namespace Drupal\mon_module\Controller;

use Drupal\Core\Controller\ControllerBase;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;
use Symfony\Component\HttpFoundation\Response;

final class ListeController extends ControllerBase {

  public function __construct(
    private readonly EntityTypeManagerInterface $entityTypeManager,
  ) {}

  public static function create(ContainerInterface $container): static {
    return new static(
      $container->get('entity_type.manager'),
    );
  }

  public function build(): array|Response {
    $storage = $this->entityTypeManager->getStorage('node');
    $nids = $storage->getQuery()
      ->accessCheck(TRUE)
      ->condition('type', 'article')
      ->condition('status', 1)
      ->sort('created', 'DESC')
      ->range(0, 10)
      ->execute();

    $nodes = $storage->loadMultiple($nids);

    return [
      '#theme' => 'item_list',
      '#items' => array_map(fn($n) => $n->label(), $nodes),
      '#cache' => [
        'tags' => ['node_list:article'],
        'contexts' => ['user.permissions'],
      ],
    ];
  }
}
```

**Règle d'or DI :** jamais `\Drupal::service()` dans une classe — toujours `__construct()` + `create()`.

---

## Accès Custom

**Approche A — `_custom_access` direct** (classe sans DI, simple) :

```php
<?php
// src/Access/MonAccess.php — classe stateless, instanciée directement par Drupal
namespace Drupal\mon_module\Access;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Access\AccessResultInterface;
use Drupal\Core\Routing\Access\AccessInterface;
use Drupal\Core\Session\AccountInterface;

final class MonAccess implements AccessInterface {

  public function checkAccess(AccountInterface $account): AccessResultInterface {
    return AccessResult::allowedIf(
      $account->hasPermission('administer site configuration') ||
      $account->hasPermission('access mon_module')
    )->cachePerUser();
  }
}
```

**Approche B — Service taggé `access_check`** (avec DI, pour logique complexe) :

```yaml
# mon_module.services.yml
mon_module.access_check:
  class: Drupal\mon_module\Access\MonAccessCheck
  arguments: ['@current_user']
  tags:
    - { name: access_check, applies_to: _mon_module_access }
```

```yaml
# mon_module.routing.yml — utiliser _mon_module_access (pas _custom_access !)
mon_module.ma_route:
  path: '/mon-module'
  defaults:
    _controller: '...'
  requirements:
    _mon_module_access: 'TRUE'   # ← correspond à applies_to du tag
```

**⚠️ Ces deux approches sont mutuellement exclusives.** `_custom_access` instancie la classe directement (pas de DI). Le tag `access_check` + `applies_to` requiert `_mon_module_access: 'TRUE'` dans le routing — jamais `_custom_access`.
```

---

## Routes avec Paramètres (upcasting)

```yaml
mon_module.detail:
  path: '/mon-module/{node}'
  defaults:
    _controller: '\Drupal\mon_module\Controller\DetailController::build'
  requirements:
    _permission: 'access content'
    node: \d+
  options:
    parameters:
      node:
        type: entity:node   # Drupal charge l'entité automatiquement
```

Le paramètre `{node}` devient directement un objet `NodeInterface` dans le Controller.

---

## Titre Dynamique (`_title_callback`)

Pour un titre qui dépend du paramètre de la route (ex: nom de l'entité) :

```yaml
mon_module.detail:
  path: '/mon-module/{node}'
  defaults:
    _controller: '\Drupal\mon_module\Controller\DetailController::build'
    _title_callback: '\Drupal\mon_module\Controller\DetailController::title'
  requirements:
    _permission: 'access content'
  options:
    parameters:
      node:
        type: entity:node
```

```php
public function title(NodeInterface $node): string {
  return $node->label();
}
```

---

## Réponses alternatives

```php
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Response;

// JSON
return new JsonResponse(['status' => 'ok', 'count' => count($nids)]);

// Redirect
return $this->redirect('mon_module.liste');

// Render array (HTML standard)
return ['#markup' => $this->t('Hello')];
```
