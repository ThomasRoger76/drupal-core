# Content Moderation — Module Core Drupal

## Activation et concepts

```bash
docker compose exec php drush en content_moderation workflows -y
```

Le module `content_moderation` s'appuie sur `workflows` pour définir des machines à états assignées à des bundles d'entités.

**Trois concepts fondamentaux :**

- **États** (`states`) — représentent une étape éditoriale : `draft`, `needs_review`, `published`, `archived`. Chaque état indique si la révision est publiée (`published: true`) et si c'est la révision par défaut (`default_revision: true`).
- **Transitions** (`transitions`) — règles de passage d'un état à un autre. Portent un label, une liste d'états source et un état cible. Contrôlées par des permissions Drupal.
- **Workflows assignés à des bundles** — un workflow est lié explicitement à `node:article`, `node:page`, etc. dans la configuration `entity_types`.

Quand un workflow est activé sur un bundle, Drupal gère automatiquement les révisions : chaque `save()` crée une nouvelle révision, ce qui préserve l'historique complet.

---

## Vérifier l'état de modération d'une entité

```php
<?php
// src/Service/ModerationChecker.php
namespace Drupal\mon_module\Service;

use Drupal\content_moderation\ModerationInformationInterface;
use Drupal\node\NodeInterface;
use Drupal\Core\Session\AccountInterface;

final class ModerationChecker {

  public function __construct(
    private readonly ModerationInformationInterface $moderationInformation,
  ) {}

  /**
   * Vérifie si la révision par défaut de l'entité est publiée.
   */
  public function estPublie(NodeInterface $node): bool {
    return $this->moderationInformation->isDefaultRevisionPublished($node);
  }

  /**
   * Retourne l'état de modération courant (ex: 'draft', 'published').
   */
  public function getEtat(NodeInterface $node): string {
    return $node->moderation_state->value;
  }

  /**
   * Vérifie si l'entité est soumise à un workflow de modération.
   */
  public function estGereeParWorkflow(NodeInterface $node): bool {
    return $this->moderationInformation->isModeratedEntity($node);
  }
}
```

```yaml
# mon_module.services.yml
services:
  mon_module.moderation_checker:
    class: Drupal\mon_module\Service\ModerationChecker
    arguments:
      - '@content_moderation.moderation_information'
```

---

## Changer l'état programmatiquement

```php
// Méthode simple — changer l'état d'un nœud
$node->set('moderation_state', 'published');
$node->save();  // Crée une nouvelle révision avec le nouvel état
```

**Pattern sécurisé avec vérification de transition valide :**

```php
<?php
// src/Service/ModerationTransitionService.php
namespace Drupal\mon_module\Service;

use Drupal\content_moderation\ModerationInformationInterface;
use Drupal\content_moderation\StateTransitionValidationInterface;
use Drupal\Core\Session\AccountInterface;
use Drupal\node\NodeInterface;

final class ModerationTransitionService {

  public function __construct(
    private readonly ModerationInformationInterface $moderationInformation,
    private readonly StateTransitionValidationInterface $stateTransitionValidation,
  ) {}

  /**
   * Tente de publier un nœud si la transition est autorisée pour ce compte.
   *
   * @return bool TRUE si la transition a eu lieu, FALSE si non autorisée.
   */
  public function publier(NodeInterface $node, AccountInterface $account): bool {
    // Récupère les transitions valides pour ce compte depuis l'état courant
    $transitions = $this->stateTransitionValidation->getValidTransitions($node, $account);

    if (!isset($transitions['publish'])) {
      return FALSE;
    }

    $node->set('moderation_state', 'published');
    $node->save();
    return TRUE;
  }

  /**
   * Transition générique avec vérification.
   */
  public function transitionner(NodeInterface $node, string $toState, AccountInterface $account): bool {
    $transitions = $this->stateTransitionValidation->getValidTransitions($node, $account);

    // Chercher la transition dont l'état cible correspond
    foreach ($transitions as $transition) {
      if ($transition->to()->id() === $toState) {
        $node->set('moderation_state', $toState);
        $node->save();
        return TRUE;
      }
    }

    return FALSE;
  }
}
```

```yaml
# mon_module.services.yml
services:
  mon_module.moderation_transition:
    class: Drupal\mon_module\Service\ModerationTransitionService
    arguments:
      - '@content_moderation.moderation_information'
      - '@content_moderation.state_transition_validation'
```

---

## WorkflowTransitionEvent — Réagir aux transitions

```php
<?php
// src/EventSubscriber/ModerationSubscriber.php
namespace Drupal\mon_module\EventSubscriber;

use Drupal\content_moderation\Event\ContentModerationStateChangedEvent;
use Drupal\mon_module\Service\NotificationService;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

final class ModerationSubscriber implements EventSubscriberInterface {

  public function __construct(
    private readonly NotificationService $notificationService,
  ) {}

  public static function getSubscribedEvents(): array {
    return [
      // Déclenché APRÈS la sauvegarde de la révision — non annulable
      ContentModerationStateChangedEvent::class => 'onStateChanged',
    ];
  }

  public function onStateChanged(ContentModerationStateChangedEvent $event): void {
    $entity = $event->getModeratedEntity();
    $from   = $event->getOriginalState();
    $to     = $event->getNewState();

    // Notification quand un article est publié
    if ($to === 'published' && $entity->getEntityTypeId() === 'node') {
      $this->notificationService->notifyPublication($entity);
    }

    // Retirer du moteur de recherche quand archivé
    if ($to === 'archived') {
      search_api_entity_delete($entity);
    }

    // Log des transitions vers la révision
    if ($from === 'draft' && $to === 'needs_review') {
      \Drupal::logger('mon_module')->info(
        'Entité @type #@id soumise pour révision.',
        ['@type' => $entity->getEntityTypeId(), '@id' => $entity->id()]
      );
    }
  }
}
```

```yaml
# mon_module.services.yml
services:
  mon_module.moderation_subscriber:
    class: Drupal\mon_module\EventSubscriber\ModerationSubscriber
    arguments:
      - '@mon_module.notification_service'
    tags:
      - { name: event_subscriber }
```

---

## hook_entity_access pour la modération

```php
<?php
// mon_module.module

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Access\AccessResultInterface;
use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Session\AccountInterface;
use Drupal\node\NodeInterface;

/**
 * Contrôle d'accès custom basé sur l'état de modération.
 */
function mon_module_entity_access(EntityInterface $entity, string $operation, AccountInterface $account): AccessResultInterface {
  if (!($entity instanceof NodeInterface)) {
    return AccessResult::neutral();
  }

  // Seul l'auteur peut modifier ses propres brouillons
  if ($operation === 'update' && $entity->moderation_state->value === 'draft') {
    return AccessResult::allowedIf(
      $entity->getOwnerId() === $account->id()
    )->cachePerUser()->addCacheableDependency($entity);
  }

  // Interdire la suppression d'un contenu publié aux non-admins
  if ($operation === 'delete' && $entity->moderation_state->value === 'published') {
    return AccessResult::forbiddenIf(
      !$account->hasPermission('administer content')
    )->cachePerPermissions()->addCacheableDependency($entity);
  }

  return AccessResult::neutral();
}
```

**Règle cache :** toujours appeler `.cachePerUser()` ou `.cachePerPermissions()` + `addCacheableDependency()` pour que le cache de rendu se invalide correctement.

---

## Requêtes EntityQuery avec révisions

```php
<?php
// Dans un service — charger des révisions et filtrer par état

// Charger la DERNIÈRE révision d'un nœud (pas forcément la révision par défaut publiée)
$storage = $this->entityTypeManager->getStorage('node');
$latest_revision_id = $storage->getLatestRevisionId($nid);

if ($latest_revision_id) {
  /** @var \Drupal\node\NodeInterface $latest */
  $latest = $storage->loadRevision($latest_revision_id);
  $etat = $latest->moderation_state->value;  // ex: 'needs_review'
}

// Charger la révision par défaut (celle visible aux visiteurs)
$default = $storage->load($nid);  // Toujours la révision par défaut

// Lister tous les nœuds en état 'needs_review'
$ids = $this->entityTypeManager->getStorage('node')
  ->getQuery()
  ->condition('moderation_state', 'needs_review')
  ->accessCheck(TRUE)
  ->execute();

$nodes = $storage->loadMultiple($ids);

// Lister tous les nœuds avec au moins une révision en draft (requête sur révisions)
$ids = $this->entityTypeManager->getStorage('node')
  ->getQuery()
  ->allRevisions()                               // Inclure toutes les révisions
  ->condition('moderation_state', 'draft')
  ->condition('uid', $this->currentUser->id())   // Brouillons de l'utilisateur courant
  ->accessCheck(TRUE)
  ->sort('changed', 'DESC')
  ->execute();
// $ids contient des paires [revision_id => nid]
```

**`getLatestRevisionId()` vs `load()`** :
- `load($nid)` — révision par défaut (publiée si workflow actif)
- `getLatestRevisionId($nid)` + `loadRevision()` — dernière révision, quel que soit l'état

---

## Configuration du workflow en YAML

```yaml
# config/install/workflows.workflow.editorial.yml
id: editorial
label: 'Workflow éditorial'
type: content_moderation
type_settings:
  states:
    draft:
      label: Brouillon
      published: false
      default_revision: false
      weight: 0
    needs_review:
      label: 'En révision'
      published: false
      default_revision: false
      weight: 1
    published:
      label: Publié
      published: true
      default_revision: true
      weight: 2
    archived:
      label: Archivé
      published: false
      default_revision: true     # La révision archivée RESTE la default (pas de dépublication sauvage)
      weight: 3
  transitions:
    create_new_draft:
      label: 'Créer un brouillon'
      from: [draft, published]
      to: draft
      weight: 0
    submit_for_review:
      label: 'Soumettre pour révision'
      from: [draft]
      to: needs_review
      weight: 1
    publish:
      label: 'Publier'
      from: [needs_review, draft]
      to: published
      weight: 2
    archive:
      label: 'Archiver'
      from: [published]
      to: archived
      weight: 3
    reject:
      label: 'Rejeter (retour brouillon)'
      from: [needs_review]
      to: draft
      weight: 4
  entity_types:
    node: [article, page]
```

**Permissions générées automatiquement par le module** pour chaque transition :
- `use editorial transition create_new_draft`
- `use editorial transition publish`
- etc.

À assigner aux rôles dans `config/install/user.role.*.yml` ou via l'interface.

---

## Views integration — Contenus par état de modération

```yaml
# Exemple de filtre Views pour lister les contenus en attente de révision
# Dans l'interface : Ajouter un filtre "Moderation state" (champ fourni par content_moderation)
# Valeur : needs_review
# Accès : permission 'view any unpublished content' ou 'view own unpublished content'
```

**En code — Vue programmatique :**

```php
// Render array basé sur une vue existante filtrée par état
$view = \Drupal::service('entity_type.manager')
  ->getStorage('view')
  ->load('content_moderation_dashboard');

// Ou directement via le service Views
$build = [
  '#type'      => 'view',
  '#name'      => 'content',
  '#display_id' => 'moderation_queue',
  '#arguments' => ['needs_review'],
];
```

**Configuration Views recommandée pour la file éditoriale :**
- Source : `Content revisions` (et non `Content`) pour voir toutes les révisions
- Filtre : `Moderation state = needs_review`
- Filtre accès : `Permission: use editorial transition publish`
- Champs : Titre (lien), Auteur, Date modif, État actuel, Lien "Modérer"

---

## Anti-patterns

```php
// ❌ Modifier moderation_state sans save() — état jamais persisté en base
$node->set('moderation_state', 'published');
// ... oubli de $node->save() → AUCUN effet

// ❌ setPublished() contourne le workflow — la révision est publiée
// mais moderation_state reste 'draft' → incohérence de données
$node->setPublished();
$node->save();

// ✅ Toujours passer par moderation_state pour respecter le workflow
$node->set('moderation_state', 'published');
$node->save();
```

```php
// ❌ load() quand on veut la dernière révision non publiée
$node = $storage->load($nid);
// → Retourne la révision par défaut (publiée), pas le brouillon en cours

// ✅ Pour le brouillon en cours
$latestId = $storage->getLatestRevisionId($nid);
$node = $storage->loadRevision($latestId);
```

```php
// ❌ Changer l'état sans vérifier les permissions de transition
$node->set('moderation_state', 'published');
$node->save();
// → Contourne les permissions Drupal sur les transitions

// ✅ Valider la transition pour le compte courant (voir ModerationTransitionService ci-dessus)
$this->moderationTransitionService->publier($node, $this->currentUser->getAccount());
```

```php
// ❌ Requête EntityQuery sur 'status' avec un workflow actif
$query->condition('status', 1);
// → status=1 ne correspond pas forcément à published dans content_moderation
// Les deux valeurs peuvent être désynchronisées si on les gère séparément

// ✅ Filtrer explicitement par moderation_state
$query->condition('moderation_state', 'published');
```
