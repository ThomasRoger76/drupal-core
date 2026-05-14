# Database API

## EntityQuery vs Database API — Quand Utiliser Quoi

| Situation | Outil | Raison |
|-----------|-------|--------|
| Requêter des entités Drupal (nodes, users, terms…) | `EntityQuery` | Respecte les permissions, cache intégré |
| Requêtes sur tes propres tables custom | Database API | EntityQuery ne connaît pas tes tables |
| JOINs complexes entre tables Drupal | Database API | EntityQuery ne supporte pas les JOINs arbitraires |
| Rapports / agrégations (GROUP BY, COUNT) | Database API | Performances, requêtes analytiques |
| Migrations de données en masse | Database API | Bypass des events/hooks Drupal intentionnel |
| Comptage rapide sans charger les entités | Database API | Plus léger qu'un EntityQuery + loadMultiple |

**Règle :** si tu peux utiliser EntityQuery, fais-le. Le Database API direct est pour les cas où EntityQuery est insuffisant.

---

## Injection du Service `database`

```php
use Drupal\Component\Datetime\TimeInterface;
use Drupal\Core\Database\Connection;
use Psr\Log\LoggerInterface;

public function __construct(
  private readonly Connection $database,
  private readonly TimeInterface $time,       // @datetime.time
  private readonly LoggerInterface $logger,   // @logger.channel.mon_module
) {}

public static function create(ContainerInterface $container): static {
  return new static(
    $container->get('database'),
    $container->get('datetime.time'),
    $container->get('logger.channel.mon_module'),
  );
}
```

Dans un hook `.module` : `$database = \Drupal::database();` / `\Drupal::time()` / `\Drupal::logger('mon_module')` sont acceptables.

---

## SELECT

```php
// Requête simple
$results = $this->database
  ->select('node_field_data', 'n')
  ->fields('n', ['nid', 'title', 'created'])
  ->condition('n.type', 'article')
  ->condition('n.status', 1)
  ->orderBy('n.created', 'DESC')
  ->range(0, 10)
  ->execute()
  ->fetchAll();

// Avec JOIN
$query = $this->database->select('node_field_data', 'n');
$query->join('node__field_categorie', 'fc', 'n.nid = fc.entity_id');
$query->fields('n', ['nid', 'title'])
  ->fields('fc', ['field_categorie_target_id'])
  ->condition('n.status', 1)
  ->condition('fc.field_categorie_target_id', [1, 2, 3], 'IN');
$results = $query->execute()->fetchAllAssoc('nid');

// Agrégation simple (COUNT)
$count = $this->database
  ->select('node_field_data', 'n')
  ->condition('type', 'article')
  ->countQuery()
  ->execute()
  ->fetchField();

// Agrégation complexe (GROUP BY, SUM, MAX…)
$query = $this->database->select('mon_module_items', 'i');
$query->addField('i', 'status');
$query->addExpression('COUNT(*)', 'total');
$query->addExpression('MAX(i.created)', 'last_created');
$query->groupBy('i.status');
$query->having('COUNT(*) > :min', [':min' => 5]);
$results = $query->execute()->fetchAllAssoc('status');

// Requête brute (éviter sauf si vraiment nécessaire)
// Les {} autour du nom de table sont OBLIGATOIRES : Drupal ajoute le préfixe DB configuré
// Ex: {node_field_data} → 'drupal_node_field_data' si préfixe = 'drupal_'
$results = $this->database
  ->query('SELECT nid, title FROM {node_field_data} WHERE type = :type', [':type' => 'article'])
  ->fetchAll();

// fetchAll() → tableau d'objets stdClass
// fetchAllAssoc('col') → tableau indexé par colonne
// fetchField() → valeur scalaire unique
// fetchCol() → tableau de valeurs d'une colonne
// fetchAssoc() → une seule ligne en tableau associatif
```

---

## INSERT / UPDATE / DELETE / UPSERT

```php
// INSERT
// Dans une classe : injecter TimeInterface ($this->time) via @datetime.time
// Dans un hook .module : \Drupal::time() est acceptable
$this->database->insert('mon_module_table')
  ->fields([
    'name'       => 'Exemple',
    'value'      => 42,
    'created_at' => $this->time->getRequestTime(),
  ])
  ->execute();

// INSERT multiple (batch)
$query = $this->database->insert('mon_module_table')
  ->fields(['name', 'value']);
foreach ($data as $row) {
  $query->values([$row['name'], $row['value']]);
}
$query->execute();

// UPDATE
$this->database->update('mon_module_table')
  ->fields(['value' => 99, 'updated_at' => $this->time->getRequestTime()])
  ->condition('name', 'Exemple')
  ->execute();

// DELETE
$this->database->delete('mon_module_table')
  ->condition('created_at', strtotime('-1 year'), '<')
  ->execute();

// UPSERT (INSERT ou UPDATE si existe)
$this->database->upsert('mon_module_table')
  ->key('name')                             // Clé d'unicité
  ->fields(['name', 'value', 'updated_at'])
  ->values(['Exemple', 42, $this->time->getRequestTime()])
  ->execute();
```

---

## Transactions

```php
$transaction = $this->database->startTransaction();
try {
  $this->database->insert('mon_module_table')
    ->fields(['name' => 'A', 'value' => 1])
    ->execute();
  $this->database->update('autre_table')
    ->fields(['status' => 'processed'])
    ->condition('id', 123)
    ->execute();
  // Commit implicite à la destruction de $transaction
} catch (\Exception $e) {
  $transaction->rollBack();
  // Dans une classe : utiliser $this->logger (LoggerInterface injecté via @logger.channel.mon_module)
  // Dans un hook : \Drupal::logger('mon_module') est acceptable
  $this->logger->error('Transaction échouée : @msg', ['@msg' => $e->getMessage()]);
  throw $e;
}
```

---

## `hook_schema()` — Définir une Table Custom

```php
// Dans mon_module.install
function mon_module_schema(): array {
  $schema['mon_module_items'] = [
    'description' => 'Stocke les items de Mon Module.',
    'fields' => [
      'id' => [
        'type'        => 'serial',
        'not null'    => TRUE,
        'description' => 'Clé primaire auto-incrémentée.',
      ],
      'nid' => [
        'type'     => 'int',
        'unsigned' => TRUE,
        'not null' => TRUE,
        'default'  => 0,
      ],
      'name' => [
        'type'     => 'varchar',
        'length'   => 255,
        'not null' => TRUE,
        'default'  => '',
      ],
      'data' => [
        'type'  => 'blob',
        'size'  => 'big',                  // tiny, small, medium, normal, big
        'not null' => FALSE,
      ],
      'status' => [
        'type'     => 'int',
        'size'     => 'tiny',
        'not null' => TRUE,
        'default'  => 0,
      ],
      'created' => [
        'type'     => 'int',
        'not null' => TRUE,
        'default'  => 0,
      ],
    ],
    'primary key' => ['id'],
    'indexes' => [
      'nid'    => ['nid'],
      'status' => ['status'],
    ],
    'unique keys' => [
      'name_nid' => ['name', 'nid'],
    ],
    'foreign keys' => [
      'node' => ['table' => 'node', 'columns' => ['nid' => 'nid']],
    ],
  ];

  return $schema;
}
```

---

## `hook_update_N()` — Migrations de Schéma

```php
// Dans mon_module.install
// Convention de numérotation : MODULE_update_XYYY
//   X   = version Drupal majeure (8, 9, 10, 11…)
//   YYY = numéro séquentiel à 3 chiffres (001, 002…)
// Exemple : mon_module_update_10001 = 1ère update pour D10

/**
 * Ajouter la colonne 'priority' à mon_module_items.
 */
function mon_module_update_10001(): void {
  $spec = [
    'type'     => 'int',
    'not null' => TRUE,
    'default'  => 0,
    'description' => 'Priorité de l\'item.',
  ];
  \Drupal::database()->schema()->addField('mon_module_items', 'priority', $spec);
}

/**
 * Migrer les données de l'ancien format.
 */
function mon_module_update_10002(array &$sandbox): string {
  // Mise à jour par batch (pour de gros volumes)
  if (!isset($sandbox['total'])) {
    $sandbox['total']    = \Drupal::database()->select('mon_module_items')->countQuery()->execute()->fetchField();
    $sandbox['progress'] = 0;
    $sandbox['limit']    = 50;
  }

  $items = \Drupal::database()
    ->select('mon_module_items', 'i')
    ->fields('i', ['id', 'data'])
    ->range($sandbox['progress'], $sandbox['limit'])
    ->execute()
    ->fetchAll();

  foreach ($items as $item) {
    // Migrer...
    $sandbox['progress']++;
  }

  $sandbox['#finished'] = $sandbox['progress'] >= $sandbox['total']
    ? 1
    : $sandbox['progress'] / $sandbox['total'];

  return "Migré {$sandbox['progress']}/{$sandbox['total']} items.";
}
```

Appliquer les updates : `ddev drush updb -y`

---

## Query Tags — Intégration avec le Système de Permissions

```php
// Ajouter des tags permettant à d'autres modules d'altérer la requête
$query = $this->database->select('mon_module_items', 'i')
  ->fields('i')
  ->addTag('mon_module_items')   // hook_query_TAG_alter() peut modifier cette requête
  ->addTag('node_access');       // Applique les restrictions d'accès aux nœuds si jointure
```

```php
// Un autre module peut altérer la requête via ce tag
function autre_module_query_mon_module_items_alter(QueryAlterableInterface $query): void {
  $query->condition('i.status', 1);
}
```
