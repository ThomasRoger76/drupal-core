# Hooks vs Event Subscribers

## Règle Fondamentale

**Hooks → fichier `.module` uniquement** (ou `.install` pour `hook_schema`, `hook_install`).
Drupal ne découvre pas les hooks dans des classes PHP — seul le fichier `.module` est scanné.

---

## Hooks Essentiels à Connaître

Les hooks vont dans `mon_module.module`. Ajouter les `use` statements correspondants en haut du fichier :

```php
// En haut de mon_module.module
use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Entity\Display\EntityViewDisplayInterface;
use Drupal\Core\Access\AccessResultInterface;
use Drupal\Core\Session\AccountInterface;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\Render\BubbleableMetadata;
use Drupal\node\NodeInterface;
use Drupal\block\Entity\Block;
use Drupal\user\UserInterface;
```

```php
// SCHÉMA & INSTALL (dans .install)
function mon_module_schema(): array { /* DDL → voir database-api.md */ }
function mon_module_install(): void { /* post-installation */ }
function mon_module_uninstall(): void { /* nettoyage */ }

// ENTITÉS
function mon_module_entity_presave(EntityInterface $entity): void {
  if ($entity->getEntityTypeId() === 'node' && $entity->bundle() === 'article') {
    // Traitement avant sauvegarde
  }
}
function mon_module_entity_insert(EntityInterface $entity): void { }
function mon_module_entity_update(EntityInterface $entity): void { }
function mon_module_entity_delete(EntityInterface $entity): void { }
function mon_module_entity_view(array &$build, EntityInterface $entity, EntityViewDisplayInterface $display, string $view_mode): void {
  // Modifier le render array d'une entité affichée
  if ($entity->getEntityTypeId() === 'node') {
    $build['#attached']['library'][] = 'mon_module/custom';
  }
}

// ACCÈS
function mon_module_node_access(NodeInterface $node, string $op, AccountInterface $account): AccessResultInterface {
  // Retourner neutral() si aucune opinion, jamais forbidden() sans raison sûre
  return AccessResult::neutral();
}
function mon_module_block_access(Block $block, string $operation, AccountInterface $account): AccessResultInterface {
  return AccessResult::neutral();
}

// FORMULAIRES
function mon_module_form_alter(array &$form, FormStateInterface $form_state, string $form_id): void { }
function mon_module_form_node_article_form_alter(array &$form, FormStateInterface $form_state): void {
  // Ciblage précis : hook_form_FORM_ID_alter — préférer cette version
}

// THÈME & RENDU
function mon_module_theme(): array {
  return [
    'mon_template' => [
      'variables' => ['items' => [], 'title' => NULL],
      'template'  => 'mon-template',               // → templates/mon-template.html.twig
    ],
  ];
}
function mon_module_preprocess_node(array &$variables): void {
  // Injecter des variables dans les templates node
  $variables['custom_date'] = $variables['node']->getCreatedTime();
}
function mon_module_preprocess_page(array &$variables): void {
  $variables['site_name'] = \Drupal::config('system.site')->get('name');
}
function mon_module_theme_suggestions_page_alter(array &$suggestions, array $variables): void {
  if ($node = \Drupal::routeMatch()->getParameter('node')) {
    $suggestions[] = 'page__node__' . $node->bundle();
  }
}

// ASSETS GLOBAUX
function mon_module_page_attachments(array &$attachments): void {
  // Ajouter CSS/JS à TOUTES les pages
  $attachments['#attached']['library'][]                           = 'mon_module/global';
  $attachments['#attached']['drupalSettings']['monModule']['key']  = 'value';
}

// UTILISATEURS
function mon_module_user_login(UserInterface $account): void { }
function mon_module_user_logout(UserInterface $account): void { }

// TOKENS
function mon_module_token_info(): array {
  return [
    'tokens' => [
      'node' => [
        'custom-token' => ['name' => t('Custom Token'), 'description' => t('Description.')],
      ],
    ],
  ];
}
function mon_module_tokens(string $type, array $tokens, array $data, array $options, BubbleableMetadata $bubbleable_metadata): array {
  $replacements = [];
  if ($type === 'node' && isset($data['node'])) {
    foreach ($tokens as $name => $original) {
      if ($name === 'custom-token') {
        $replacements[$original] = 'valeur custom';
      }
    }
  }
  return $replacements;
}

// VUES
function mon_module_views_data(): array { /* exposer des tables custom à Views */ return []; }
function mon_module_views_data_alter(array &$data): void { }

// CRON
function mon_module_cron(): void {
  $state = \Drupal::state();
  $state->set('mon_module.last_cron', \Drupal::time()->getRequestTime());
}

// MENU / LIENS
function mon_module_menu_links_discovered_alter(array &$links): void { }
```

### Hook D11 — `#[Hook]` Attribute dans une Classe

Depuis Drupal 11, les hooks peuvent être déclarés dans des classes PHP, avec DI complète :

```php
<?php
// src/Hook/MonModuleHooks.php
namespace Drupal\mon_module\Hook;

use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Hook\Attribute\Hook;
use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Form\FormStateInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

class MonModuleHooks {

  public function __construct(
    private readonly EntityTypeManagerInterface $entityTypeManager,
  ) {}

  // ✅ AUTODISCOVERY D11 — PAS de services.yml requis pour les hooks
  // Drupal découvre les classes dans src/Hook/ via PSR-4 automatiquement.
  // Le container est reconstruit à chaque drush cr et détecte les #[Hook].
  //
  // ⚠️ EXCEPTION : si la classe a besoin de DI (services injectés),
  // elle DOIT être déclarée comme service dans mon_module.services.yml :
  //   mon_module.hooks:
  //     class: Drupal\mon_module\Hook\MonModuleHooks
  //     arguments: ['@entity_type.manager']
  //     tags:
  //       - { name: drupal.hook_implementations }
  //
  // Sans DI : la classe peut vivre dans src/Hook/ sans services.yml

  #[Hook('entity_presave')]
  public function entityPresave(EntityInterface $entity): void {
    if ($entity->getEntityTypeId() !== 'node') {
      return;
    }
    // Accès aux services injectés : $this->entityTypeManager
    $storage = $this->entityTypeManager->getStorage('taxonomy_term');
    // ...
  }

  #[Hook('form_node_article_form_alter')]
  public function formNodeArticleFormAlter(array &$form, FormStateInterface $form_state): void {
    $form['field_date']['#required'] = TRUE;
  }
}
```

**Avantages vs .module :** DI via constructeur, testable en isolation (PHPUnit Unit), groupement logique des hooks, IDE-friendly (navigation, autocompletion).

---

## Hook vs EventSubscriber — Tableau Décisionnel

| Situation | Utiliser | Raison |
|-----------|----------|--------|
| Modifier un formulaire existant | `hook_form_alter` | Pas d'équivalent Event |
| Réagir avant/après sauvegarde entité | `hook_entity_presave/insert` | Standard Drupal |
| Ajouter des données à Views | `hook_views_data` | Pas d'équivalent Event |
| Modifier la réponse HTTP (headers…) | `EventSubscriber` (KernelEvents) | Symfony natif |
| Rediriger selon condition sur la requête | `EventSubscriber` (KernelEvents::REQUEST) | Intercepte avant le routing |
| Réagir à une transition de Workflow | `EventSubscriber` | Symfony Event Dispatcher |
| S'abonner aux événements d'un module contrib | `EventSubscriber` | Si le module dispatche des events |
| Besoin de DI propre dans la logique | `EventSubscriber` (service) | Injection via constructeur |
| Code rapide, couplage Drupal assumé | Hook dans `.module` | Plus simple |

---

## EventSubscriber — Pattern Complet

```php
<?php
// src/EventSubscriber/MonSubscriber.php
namespace Drupal\mon_module\EventSubscriber;

use Drupal\Core\Session\AccountProxyInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\RedirectResponse;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\Event\ResponseEvent;
use Symfony\Component\HttpKernel\KernelEvents;

final class MonSubscriber implements EventSubscriberInterface {

  public function __construct(
    private readonly AccountProxyInterface $currentUser,
  ) {}

  public static function getSubscribedEvents(): array {
    return [
      KernelEvents::REQUEST => [['onRequest', 20]],  // priorité 20
      KernelEvents::RESPONSE => [['onResponse', 0]],
    ];
  }

  public function onRequest(RequestEvent $event): void {
    if (!$event->isMainRequest()) {
      return; // Ignorer les sous-requêtes
    }
    $request = $event->getRequest();
    // Exemple : rediriger les anonymes hors de /admin
    if ($this->currentUser->isAnonymous() && str_starts_with($request->getPathInfo(), '/admin')) {
      $event->setResponse(new RedirectResponse('/user/login'));
    }
  }

  public function onResponse(\Symfony\Component\HttpKernel\Event\ResponseEvent $event): void {
    $event->getResponse()->headers->set('X-Mon-Module', '1.0');
  }
}
```

```yaml
# mon_module.services.yml
services:
  mon_module.mon_subscriber:
    class: Drupal\mon_module\EventSubscriber\MonSubscriber
    arguments: ['@current_user']
    tags:
      - { name: event_subscriber }
```

---

## Priorités des Événements

Les priorités s'expriment en entier dans `getSubscribedEvents()` :
- `+999` → s'exécute **en premier** (avant tout)
- `0` → priorité **neutre** (défaut)
- `-999` → s'exécute **en dernier** (après tout)

```php
public static function getSubscribedEvents(): array {
  return [
    KernelEvents::REQUEST => [
      ['checkAuthentication', 300],  // Avant la plupart des subscribers
      ['logRequest', -100],          // Après les autres
    ],
  ];
}
```

Pour les hooks, l'ordre s'ajuste via `hook_module_implements_alter()`.

---

## Événements Courants Drupal/Symfony

| Événement | Classe | Usage |
|-----------|--------|-------|
| `kernel.request` | `KernelEvents::REQUEST` | Authentification, redirections |
| `kernel.response` | `KernelEvents::RESPONSE` | Modifier headers, caching |
| `kernel.exception` | `KernelEvents::EXCEPTION` | Gérer les erreurs 404/403 |
| `routing.route_alter` | `RoutingEvents::ALTER` | Modifier les routes au runtime |
| `workflow.guard` | `Symfony\Component\Workflow\Event\GuardEvent` | Bloquer une transition (module `workflows`) |
| `workflow.completed` | `Symfony\Component\Workflow\Event\CompletedEvent` | Après transition confirmée |

---

## Queue API — Traitement Asynchrone

La Queue API permet de différer un traitement lourd (import CSV, envoi d'emails, appels API externes) pour l'exécuter via cron ou manuellement.

### Ajouter des items à la queue

```php
// Dans un Controller, Form ou service
$queue = \Drupal::queue('mon_module_traitement');
foreach ($lignes_csv as $ligne) {
  $queue->createItem($ligne);
}
// L'item est sérialisé automatiquement
```

### Définir un QueueWorker (Plugin)

```php
// src/Plugin/QueueWorker/TraitementImport.php
namespace Drupal\mon_module\Plugin\QueueWorker;

use Drupal\Core\Queue\QueueWorkerBase;

// D10- : annotation
// #[QueueWorker(id: "mon_module_traitement", title: "Traitement Import", cron: ["time" => 60])]  // D11+
/**
 * @QueueWorker(
 *   id = "mon_module_traitement",
 *   title = @Translation("Traitement Import"),
 *   cron = {"time" = 60}
 * )
 */
class TraitementImport extends QueueWorkerBase {

  public function processItem($data): void {
    // Traiter $data (l'item sérialisé)
    // Si une exception est levée → l'item reste dans la queue
    // Si RequeueException → requeue immédiat
    // Si SuspendQueueException → arrête le traitement de la queue
    $this->importerLigne($data);
  }
}
```

### Déclencher la queue manuellement

```bash
# Via Drush
drush queue:list                          # Voir les queues disponibles
drush queue:run mon_module_traitement     # Vider une queue spécifique
drush cron                                # Exécuter toutes les queues (cron)
```

---

## Batch API — Traitement par Lots avec Progression

Le Batch API affiche une barre de progression et évite les timeouts PHP.

```php
// Dans un formulaire ou controller — définir le batch
function mon_module_lancer_import(array $items): void {
  $operations = [];
  foreach (array_chunk($items, 50) as $chunk) {
    $operations[] = ['\Drupal\mon_module\Batch\ImportBatch::processer', [$chunk]];
  }

  $batch = [
    'title' => t('Import en cours...'),
    'operations' => $operations,
    'finished' => '\Drupal\mon_module\Batch\ImportBatch::termine',
    'error_message' => t('Une erreur est survenue.'),
  ];

  batch_set($batch);
  // batch_set() déclare le batch — la redirection vers la page de progression
  // est gérée AUTOMATIQUEMENT par Drupal selon le contexte d'appel
}

// ─── Dans un FormBase::submitForm() — context formulaire ────────────────────
public function submitForm(array &$form, FormStateInterface $form_state): void {
  $items = $this->chargerItems();
  mon_module_lancer_import($items);
  // submitForm ne retourne RIEN — Drupal détecte batch_set() et redirige
  // automatiquement vers la page de progression (/batch)
  // Pas besoin de return batch_process() dans un formulaire standard
}

// ─── Dans un Controller — context non-formulaire ─────────────────────────────
public function importPage(): array|Response {
  $items = $this->chargerItems();
  batch_set([
    'title' => t('Import...'),
    'operations' => [[[MonBatch::class, 'process'], [$items]]],
    'finished' => [MonBatch::class, 'finished'],
  ]);
  // Dans un Controller, batch_process() retourne une RedirectResponse
  return batch_process(Url::fromRoute('mon_module.import_result'));
}
```

```php
// src/Batch/ImportBatch.php
namespace Drupal\mon_module\Batch;

class ImportBatch {

  public static function processer(array $chunk, array &$context): void {
    if (!isset($context['sandbox']['total'])) {
      $context['sandbox']['total'] = count($chunk);
      $context['sandbox']['current'] = 0;
    }

    foreach ($chunk as $item) {
      // Traitement de chaque item
      \Drupal::entityTypeManager()
        ->getStorage('node')
        ->create(['type' => 'article', 'title' => $item['titre']])
        ->save();

      $context['sandbox']['current']++;
      $context['results'][] = $item['id'];
    }

    $context['message'] = t('Traitement : @current sur @total', [
      '@current' => $context['sandbox']['current'],
      '@total' => $context['sandbox']['total'],
    ]);
    $context['finished'] = $context['sandbox']['current'] / $context['sandbox']['total'];
  }

  public static function termine(bool $success, array $results, array $operations): void {
    if ($success) {
      \Drupal::messenger()->addStatus(t('@count éléments importés.', ['@count' => count($results)]));
    } else {
      \Drupal::messenger()->addError(t('Erreur pendant le batch.'));
    }
  }
}
```

---

## Drush Custom Commands

Les modules custom avec traitement de données définissent souvent une commande Drush.

### Structure

```
mon_module/
├── drush.services.yml        # Déclarer la commande comme service Drush
└── src/
    └── Commands/
        └── MonModuleCommands.php
```

```yaml
# drush.services.yml
services:
  mon_module.commands:
    class: \Drupal\mon_module\Commands\MonModuleCommands
    arguments: ['@entity_type.manager', '@queue']
    tags:
      - { name: drush.command }
```

```php
// src/Commands/MonModuleCommands.php
namespace Drupal\mon_module\Commands;

use Drush\Commands\DrushCommands;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Queue\QueueInterface;

class MonModuleCommands extends DrushCommands {

  public function __construct(
    private EntityTypeManagerInterface $entityTypeManager,
    private QueueInterface $queue,
  ) {}

  /**
   * Lancer l'import depuis la ligne de commande.
   *
   * @command mon-module:import
   * @aliases mmi
   * @option limit Nombre maximum d'items à importer
   * @usage mon-module:import --limit=100
   */
  public function import(array $options = ['limit' => 0]): void {
    $this->output()->writeln('Démarrage de l\'import...');
    $limit = (int) $options['limit'];
    // Logique d'import...
    $this->logger()->success(dt('Import terminé.'));
  }
}
```

```bash
# Utilisation
drush mon-module:import
drush mmi --limit=500
```
