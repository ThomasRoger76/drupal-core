# Services & API Interne

## Dependency Injection — Rappel

Dans une **classe** (Controller, Plugin, Service, EventSubscriber) :
```php
// TOUJOURS via constructeur + create()
public function __construct(
  private readonly EntityTypeManagerInterface $entityTypeManager,
  private readonly ConfigFactoryInterface $configFactory,
  private readonly StateInterface $state,
) {}

public static function create(ContainerInterface $container): static {
  return new static(
    $container->get('entity_type.manager'),
    $container->get('config.factory'),
    $container->get('state'),
  );
}
```

Dans un **hook** (fichier `.module`) — seul cas où `\Drupal::` est acceptable :
```php
function mon_module_cron(): void {
  $state = \Drupal::state();
  $state->set('mon_module.last_cron', \Drupal::time()->getRequestTime());
}
```

---

## Entity API — CRUD

```php
$storage = $this->entityTypeManager->getStorage('node');

// Charger une entité
$node = $storage->load(42);

// Charger plusieurs entités
$nodes = $storage->loadMultiple([1, 2, 3]);

// Créer une entité (injecter AccountProxyInterface $currentUser dans le constructeur)
$node = $storage->create([
  'type'   => 'article',
  'title'  => 'Mon titre',
  'status' => 1,
  'uid'    => $this->currentUser->id(),   // AccountProxyInterface injecté, jamais \Drupal::
]);
$node->set('field_categorie', $term_id);
$node->save();

// Modifier une entité
$node = $storage->load(42);
$node->set('title', 'Nouveau titre');
$node->set('field_date', '2026-01-15');
$node->save();

// Supprimer
$storage->load(42)?->delete();

// Charger par propriété (simple, sans EntityQuery)
$nodes = $storage->loadByProperties(['type' => 'article', 'status' => 1]);
```

---

## EntityQuery — Requêtes Complexes

```php
$storage = $this->entityTypeManager->getStorage('node');

$query = $storage->getQuery()
  ->accessCheck(TRUE)           // Respecter les permissions utilisateur
  ->condition('type', 'article')
  ->condition('status', 1)
  ->condition('field_categorie', [1, 2, 3], 'IN')
  ->condition('created', strtotime('-30 days'), '>=')
  ->sort('created', 'DESC')
  ->range(0, 20);               // LIMIT 20

// Conditions OR
$group = $query->orConditionGroup()
  ->condition('title', '%drupal%', 'LIKE')
  ->condition('field_tags.entity.name', 'drupal');
$query->condition($group);

$nids = $query->execute();
$nodes = $storage->loadMultiple($nids);
```

**`accessCheck(FALSE)`** — uniquement pour les opérations internes (cron, migrations) où les permissions ne s'appliquent pas.

---

## Config API vs State API

| | **Config API** | **State API** |
|--|---------------|--------------|
| **Service** | `config.factory` | `state` |
| **Exportable YAML** | ✅ oui (`drush cex`) | ❌ non (DB seulement) |
| **Versionnable git** | ✅ oui | ❌ non |
| **Use case** | Settings admin, features | Runtime data, timestamps cron |
| **Exemple** | Nombre d'items, clé API | Dernier run cron, cache busting |

```php
// CONFIG API — settings exportables
$config = $this->configFactory->get('mon_module.settings');
$max    = $config->get('max_items');  // lecture

$config = $this->configFactory->getEditable('mon_module.settings');
$config->set('max_items', 25)->save();  // écriture

// STATE API — données runtime volatiles
$last_run = $this->state->get('mon_module.last_cron', 0);            // défaut = 0
$this->state->set('mon_module.last_cron', $this->time->getRequestTime()); // TimeInterface injecté
$this->state->delete('mon_module.last_cron');
```

Fichier de config par défaut : `config/install/mon_module.settings.yml`
```yaml
# config/install/mon_module.settings.yml
max_items: 10
```

**⚠️ Ne jamais stocker de secrets (clés API, mots de passe) dans Config API.** Ces fichiers YAML sont versionnés en git et exportables. Les secrets vont dans `settings.php` (local) ou un vault d'environnement.

---

## Cache API — Les 3 Composantes

Le cache Drupal est **invalide automatiquement** si les tags sont bien définis. Ne pas oublier = contenu périmé.

### Tags — Invalider quand une entité change

```php
// Tags standards
'node:42'          // Ce nœud spécifique
'node_list'        // Toute liste de nœuds
'node_list:article' // Listes d'articles spécifiquement
'user:5'           // Cet utilisateur
'taxonomy_term:12' // Ce terme de taxonomie
'config:mon_module.settings'  // Cette config

// Dans un render array
$build['#cache']['tags'] = Cache::mergeTags(
  ['node_list:article'],
  $node->getCacheTags(),        // Tags de l'entité chargée
);
```

### Contexts — Varier le cache selon le contexte

```php
$build['#cache']['contexts'] = [
  'user.roles',        // Différent par rôle
  'user',              // Différent par utilisateur (granulaire)
  'url',               // Différent par URL complète
  'url.query_args',    // Différent par paramètres GET
  'languages:language_interface',  // Différent par langue
  'session',           // Différent par session (éviter si possible)
];
```

### Invalider manuellement

```php
use Drupal\Core\Cache\Cache;

// Invalider un ou plusieurs tags immédiatement (ex: après une action admin)
Cache::invalidateTags(['node_list:article', 'config:mon_module.settings']);
Cache::invalidateTags($node->getCacheTags());
```

Utiliser quand tu modifies des données hors du cycle normal save/delete d'une entité Drupal.

### Max-age — TTL

```php
use Drupal\Core\Cache\Cache;

$build['#cache']['max-age'] = Cache::PERMANENT;  // Jamais expiré (défaut recommandé avec tags)
$build['#cache']['max-age'] = 0;                  // Jamais mis en cache (à éviter)
$build['#cache']['max-age'] = 3600;               // Expire après 1h (sans tags = dangereux)
```

### Exemple Complet — Controller avec Cache

```php
public function build(): array {
  $config  = $this->configFactory->get('mon_module.settings');
  $max     = $config->get('max_items') ?? 10;
  $storage = $this->entityTypeManager->getStorage('node');

  $nids  = $storage->getQuery()
    ->accessCheck(TRUE)
    ->condition('type', 'article')
    ->condition('status', 1)
    ->sort('created', 'DESC')
    ->range(0, $max)
    ->execute();

  $nodes = $storage->loadMultiple($nids);
  $tags  = Cache::mergeTags(
    ['node_list:article', 'config:mon_module.settings'],
    array_reduce($nodes, fn($carry, $n) => Cache::mergeTags($carry, $n->getCacheTags()), []),
  );

  return [
    '#theme'  => 'item_list',
    '#items'  => array_map(fn($n) => $n->toLink()->toString(), $nodes),
    '#cache'  => [
      'tags'     => $tags,
      'contexts' => ['user.permissions', 'languages:language_interface'],
      'max-age'  => Cache::PERMANENT,
    ],
  ];
}
```

---

## Créer Son Propre Service

```yaml
# mon_module.services.yml
# ⚠️ Convention OBLIGATOIRE : identifiants en snake_case complet
# ❌ mon_module.MyService   (CamelCase → non standard, difficile à débugger)
# ✅ mon_module.my_service  (snake_case → standard Drupal)

services:

  # Service custom simple
  mon_module.article_manager:
    class: Drupal\mon_module\Service\ArticleManager
    arguments:
      - '@entity_type.manager'
      - '@current_user'
      - '@logger.channel.mon_module'   # Injecte LoggerInterface directement (PSR-3)
      - '@datetime.time'               # Injecte TimeInterface

  # Event Subscriber
  mon_module.mon_subscriber:
    class: Drupal\mon_module\EventSubscriber\MonSubscriber
    arguments: ['@current_user']
    tags:
      - { name: event_subscriber }

  # Access Check custom (pour _custom_access dans routing.yml)
  mon_module.access_check:
    class: Drupal\mon_module\Access\MonAccessCheck
    tags:
      - { name: access_check, applies_to: _mon_module_access }
```

```php
<?php
// src/Service/ArticleManager.php
namespace Drupal\mon_module\Service;

use Drupal\Component\Datetime\TimeInterface;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Session\AccountProxyInterface;
use Psr\Log\LoggerInterface;

final class ArticleManager {

  public function __construct(
    private readonly EntityTypeManagerInterface $entityTypeManager,
    private readonly AccountProxyInterface $currentUser,
    private readonly LoggerInterface $logger,        // via @logger.channel.mon_module
    private readonly TimeInterface $time,            // via @datetime.time
  ) {}

  public function getRecentArticles(int $limit = 10): array {
    $start = $this->time->getRequestTime();  // TimeInterface en usage réel
    $storage = $this->entityTypeManager->getStorage('node');

    $nids = $storage->getQuery()
      ->accessCheck(TRUE)
      ->condition('type', 'article')
      ->condition('status', 1)
      ->sort('created', 'DESC')
      ->range(0, $limit)
      ->execute();

    $articles = $storage->loadMultiple($nids);

    $this->logger->info('@count articles chargés par uid @uid (ts: @ts)', [
      '@count' => count($articles),
      '@uid'   => $this->currentUser->id(),
      '@ts'    => $start,
    ]);

    return $articles;
  }
}
```

**Règle :** les services déclarés dans `.services.yml` sont des singletons dans le container — **ne pas créer avec `new`**, toujours injecter.

---

## Anti-Patterns dans les Services — `echo` et `\Drupal::`

```php
// ❌ MAUVAISES PRATIQUES dans un service Drupal

final class MonImportService {

  public function __construct(
    private readonly EntityTypeManagerInterface $entityTypeManager,
    // Logger PAS injecté — anti-pattern
  ) {}

  public function process(): void {
    // ❌ echo dans un service — output non contrôlable, casse en contexte non-CLI
    echo "Traitement en cours...\n";

    // ❌ \Drupal:: dans une classe avec DI partielle — incohérent
    \Drupal::logger('mon_module')->notice('...');

    // ❌ \Drupal:: alors que EntityTypeManager est déjà injecté
    $storage = \Drupal::entityTypeManager()->getStorage('node');
  }
}

// ✅ CORRECT — DI complète, pas d'echo, logger injecté

final class MonImportService {

  public function __construct(
    private readonly EntityTypeManagerInterface $entityTypeManager,
    private readonly LoggerInterface $logger,            // @logger.channel.mon_module
  ) {}

  public function process(): array {  // Retourner les résultats, pas les afficher
    $storage = $this->entityTypeManager->getStorage('node');

    $this->logger->notice('Traitement en cours...');

    return ['total' => 0, 'updated' => 0, 'errors' => 0];  // Résultats structurés
  }
}
```

**Pour les Drush Commands** qui ont besoin d'output : utiliser `$this->output()->writeln()` dans la Command, pas `echo` dans le service.

---

## Drush Commands Custom

```php
<?php
// src/Commands/MonModuleCommands.php
namespace Drupal\mon_module\Commands;

use Drupal\mon_module\Service\MonImportService;
use Drush\Commands\DrushCommands;

final class MonModuleCommands extends DrushCommands {

  public function __construct(
    private readonly MonImportService $importService,
  ) {
    parent::__construct();
  }

  /**
   * Lance l'import depuis la source externe.
   *
   * @command mon-module:import
   * @aliases mmi
   * @option force Forcer la réimportation même si inchangé
   * @usage drush mon-module:import --force
   */
  public function import(array $options = ['force' => FALSE]): void {
    $results = $this->importService->process();

    $this->output()->writeln('Import terminé.');
    $this->output()->writeln("Total : {$results['total']} | Mis à jour : {$results['updated']} | Erreurs : {$results['errors']}");
  }
}
```

```yaml
# drush.services.yml — OBLIGATOIRE pour enregistrer la commande Drush
services:
  mon_module.commands:
    class: Drupal\mon_module\Commands\MonModuleCommands
    arguments: ['@mon_module.import_service']
    tags:
      - { name: drush.command }
```

---

## Batch API — Import en Masse

```php
<?php
// src/Batch/MonImportBatch.php
namespace Drupal\mon_module\Batch;

final class MonImportBatch {

  /**
   * Traite un item du batch.
   */
  public static function processItem(array $itemData, bool $force, array &$context): void {
    $context['results']['total'] = ($context['results']['total'] ?? 0) + 1;

    try {
      $mapper = \Drupal::service('mon_module.mapper');
      $result = $mapper->map($itemData, $force);
      $result ? $context['results']['updated']++ : $context['results']['skipped']++;
    }
    catch (\Exception $e) {
      $context['results']['errors'][] = $e->getMessage();
      \Drupal::logger('mon_module')->error('Erreur batch : @msg', ['@msg' => $e->getMessage()]);
    }

    $context['message'] = t('Traitement @current sur @total...', [
      '@current' => $context['results']['total'],
      '@total'   => count($context['sandbox']['items'] ?? []),
    ]);
  }

  /**
   * Callback finale du batch.
   */
  public static function finished(bool $success, array $results, array $operations): void {
    if ($success) {
      \Drupal::messenger()->addStatus(t(
        'Import terminé : @total traités, @updated mis à jour, @err erreurs.',
        [
          '@total'   => $results['total'] ?? 0,
          '@updated' => $results['updated'] ?? 0,
          '@err'     => count($results['errors'] ?? []),
        ]
      ));
    } else {
      \Drupal::messenger()->addError(t('Erreur pendant le batch.'));
    }
  }
}

// Lancer le batch depuis un formulaire ou un service
public function startBatch(array $items): void {
  $operations = array_map(
    fn($item) => [[MonImportBatch::class, 'processItem'], [$item, FALSE]],
    $items
  );

  batch_set([
    'title'            => t('Import en cours...'),
    'operations'       => $operations,
    'finished'         => [MonImportBatch::class, 'finished'],
    'init_message'     => t('Démarrage...'),
    'progress_message' => t('Traitement @current sur @total...'),
  ]);
}
```

---

## DTO — Data Transfer Object

Pattern courant pour les modules d'import/synchro : séparer la donnée externe (DTO) de l'entité Drupal.

```php
<?php
// src/DTO/ItemDTO.php
namespace Drupal\mon_module\DTO;

final class ItemDTO {

  public string $externalId;
  public string $name;
  public ?string $status;

  private function __construct() {}

  public static function fromArray(array $data): self {
    $dto = new self();
    $dto->externalId = (string) ($data['id'] ?? '');
    $dto->name       = (string) ($data['name'] ?? '');
    $dto->status     = $data['status'] ?? NULL;
    return $dto;
  }

  public function toArray(): array {
    return [
      'external_id' => $this->externalId,
      'name'        => $this->name,
      'status'      => $this->status,
    ];
  }

  public function isValid(): bool {
    return !empty($this->externalId) && !empty($this->name);
  }
}
```

**Pattern complet :** `API Response → DTO (parsing) → Model (métier) → Entity (Drupal)`

---

## Services Courants à Injecter

| Service ID | Interface | Usage |
|-----------|-----------|-------|
| `entity_type.manager` | `EntityTypeManagerInterface` | Charger/requêter entités |
| `config.factory` | `ConfigFactoryInterface` (**pas** `ConfigFactory`) | Config API |
| `state` | `StateInterface` | State API |
| `current_user` | `AccountProxyInterface` | Utilisateur courant |
| `messenger` | `MessengerInterface` | Messages flash |
| `logger.channel.mon_module` | `LoggerInterface` (PSR-3) | Logs — **recommandé** : channel dédié au module |
| `logger.factory` | `LoggerChannelFactoryInterface` | Logs factory — appeler `->get('mon_module')` |
| `language_manager` | `LanguageManagerInterface` | Langue active |
| `renderer` | `RendererInterface` | Render arrays → HTML |
| `path_alias.repository` | `AliasRepositoryInterface` | Alias d'URL (D10+ — `AliasManagerInterface` deprecated) |
| `database` | `Connection` | Requêtes SQL directes |
| `cache.default` | `CacheBackendInterface` | Cache manuel (données génériques) |
| `cache.render` | `CacheBackendInterface` | Cache render arrays HTML |
| `cache.data` | `CacheBackendInterface` | Cache données structurées |
| `datetime.time` | `TimeInterface` | Temps courant (préférer à `time()`) |
| `queue` | `QueueFactory` | Files d'attente asynchrones |
