# Layout Builder API — Drupal Core (D8.7+)

## Vue d'ensemble

Layout Builder est un module core depuis Drupal 8.7. Il permet de construire visuellement les mises en page par type de contenu (layouts par défaut) ou par nœud individuel (overrides). L'API PHP couvre l'activation, la manipulation de sections/composants et les tests.

---

## 1. Activer Layout Builder sur un type de contenu

```php
// Dans hook_install() ou hook_update_N()
use Drupal\layout_builder\Plugin\SectionStorage\OverridesSectionStorage;

/**
 * Activer Layout Builder sur le display "default" du type "article".
 */
function mon_module_install(): void {
  $display = \Drupal::entityTypeManager()
    ->getStorage('entity_view_display')
    ->load('node.article.default');

  if ($display) {
    $display
      ->enableLayoutBuilder()
      ->setOverridable()  // Permettre les overrides par nœud
      ->save();
  }
}

// Vérifier si Layout Builder est actif avant d'agir
$display = \Drupal::entityTypeManager()
  ->getStorage('entity_view_display')
  ->load('node.article.default');

if ($display && $display->isLayoutBuilderEnabled()) {
  // Layout Builder est actif sur ce display
  $est_overridable = $display->isOverridable();
}
```

---

## 2. Ajouter un bloc programmatiquement à une section Layout Builder

```php
use Drupal\layout_builder\Section;
use Drupal\layout_builder\SectionComponent;

// Créer une section avec un layout 2 colonnes
$section = new Section('layout_twocol_section', [
  'column_widths' => '50-50',
  'label'         => '',
]);

// Ajouter un bloc custom à la région "first" (colonne gauche)
$component_left = new SectionComponent(
  \Drupal::service('uuid')->generate(),
  'first',    // Région du layout (first | second | content | top | bottom)
  [
    'id'            => 'mon_module_mon_bloc',  // Plugin ID du bloc
    'label'         => 'Mon Bloc',
    'label_display' => FALSE,
    'provider'      => 'mon_module',
  ]
);
$section->appendComponent($component_left);

// Ajouter un deuxième bloc dans la colonne droite
$component_right = new SectionComponent(
  \Drupal::service('uuid')->generate(),
  'second',
  [
    'id'            => 'mon_module_autre_bloc',
    'label'         => 'Autre Bloc',
    'label_display' => '0',
    'provider'      => 'mon_module',
  ]
);
$section->appendComponent($component_right);

// Ajouter la section au display par défaut
$display = \Drupal::entityTypeManager()
  ->getStorage('entity_view_display')
  ->load('node.article.default');

if ($display && $display->isLayoutBuilderEnabled()) {
  $display->appendSection($section);
  $display->save();
}
```

---

## 3. Accéder aux sections d'un nœud avec override

```php
use Drupal\layout_builder\LayoutEntityHelperTrait;
use Drupal\node\NodeInterface;

class MonService {
  use LayoutEntityHelperTrait;

  /**
   * Retourne les sections Layout Builder d'un nœud.
   *
   * @return \Drupal\layout_builder\Section[]
   */
  public function getSections(NodeInterface $node): array {
    if (!$this->isLayoutCompatible($node)) {
      return [];
    }
    // Récupérer les sections de l'override du nœud (ou du display par défaut)
    $storage = $this->getSectionStorageForEntity($node);
    return $storage ? $storage->getSections() : [];
  }

  /**
   * Récupère tous les UUIDs de composants pour un nœud donné.
   *
   * @return string[]
   */
  public function getComponentUuids(NodeInterface $node): array {
    $uuids = [];
    foreach ($this->getSections($node) as $section) {
      foreach ($section->getComponents() as $component) {
        $uuids[] = $component->getUuid();
      }
    }
    return $uuids;
  }
}
```

---

## 4. Vérifier si un nœud a un layout personnalisé (override)

```php
use Drupal\layout_builder\LayoutEntityHelperTrait;
use Drupal\node\NodeInterface;

class MonService {
  use LayoutEntityHelperTrait;

  public function hasCustomLayout(NodeInterface $node): bool {
    // Toujours vérifier que LB est actif d'abord
    if (!$this->isLayoutCompatible($node)) {
      return FALSE;
    }
    $section_storage = $this->getSectionStorageForEntity($node);
    if (!$section_storage) {
      return FALSE;
    }
    // isOverridden() = TRUE si le nœud a ses propres sections différentes du display par défaut
    return $section_storage->isOverridden();
  }

  public function resetToDefault(NodeInterface $node): void {
    if ($this->hasCustomLayout($node)) {
      $section_storage = $this->getSectionStorageForEntity($node);
      // Supprimer les overrides du nœud — retour au layout par défaut
      $section_storage->removeSection(0);
      $section_storage->save();
    }
  }
}
```

---

## 5. Désactiver Layout Builder pour un type de contenu via code

```php
// Dans hook_update_N()
function mon_module_update_9001(): void {
  $display = \Drupal::entityTypeManager()
    ->getStorage('entity_view_display')
    ->load('node.page.default');

  if ($display && $display->isLayoutBuilderEnabled()) {
    $display->disableLayoutBuilder()->save();
  }
}

// Désactiver les overrides sans désactiver Layout Builder
function mon_module_update_9002(): void {
  $display = \Drupal::entityTypeManager()
    ->getStorage('entity_view_display')
    ->load('node.article.default');

  if ($display && $display->isLayoutBuilderEnabled()) {
    // Désactiver les overrides par nœud, garder le layout par défaut
    $display->setOverridable(FALSE)->save();
  }
}
```

---

## 6. EventSubscriber sur Layout Builder

```php
use Drupal\layout_builder\Event\SectionComponentBuildRenderArrayEvent;
use Drupal\layout_builder\LayoutBuilderEvents;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

// src/EventSubscriber/LayoutBuilderSubscriber.php
#[\Drupal\Core\Hook\Attribute\EventSubscriber]
class LayoutBuilderSubscriber implements EventSubscriberInterface {

  public static function getSubscribedEvents(): array {
    return [
      LayoutBuilderEvents::SECTION_COMPONENT_BUILD_RENDER_ARRAY => [
        ['onBuildRender', 100],
      ],
    ];
  }

  /**
   * Modifie le render array d'un composant Layout Builder.
   */
  public function onBuildRender(SectionComponentBuildRenderArrayEvent $event): void {
    $build = $event->getBuild();

    // Ajouter une classe CSS wrapper à tous les composants
    $build['#attributes']['class'][] = 'lb-component-wrapper';

    // Accéder au composant pour des modifications conditionnelles
    $component = $event->getComponent();
    $plugin_id = $component->getPluginId();

    if (str_starts_with($plugin_id, 'mon_module_')) {
      $build['#attributes']['data-module'] = 'mon-module';
    }

    $event->setBuild($build);
  }
}
```

```yaml
# mon_module.services.yml
services:
  mon_module.layout_builder_subscriber:
    class: Drupal\mon_module\EventSubscriber\LayoutBuilderSubscriber
    tags:
      - { name: event_subscriber }
```

---

## 7. Invalider le cache après modification d'une section

```php
use Drupal\Core\Cache\Cache;

// Layout Builder cache les rendus agressivement.
// Invalider après toute modification programmatique.

public function updateNodeLayout(NodeInterface $node, Section $section): void {
  $section_storage = $this->getSectionStorageForEntity($node);
  if ($section_storage) {
    $section_storage->appendSection($section);
    $section_storage->save();

    // Invalider le cache du nœud et du display
    Cache::invalidateTags([
      'node:' . $node->id(),
      'config:core.entity_view_display.node.' . $node->bundle() . '.default',
    ]);
  }
}
```

---

## 8. Tester Layout Builder (KernelTest)

```php
namespace Drupal\Tests\mon_module\Kernel;

use Drupal\KernelTests\KernelTestBase;
use Drupal\node\Entity\NodeType;

/**
 * @group mon_module
 * @group layout_builder
 */
class LayoutBuilderTest extends KernelTestBase {

  protected static $modules = [
    'layout_builder',
    'layout_discovery',
    'node',
    'user',
    'system',
    'field',
  ];

  protected function setUp(): void {
    parent::setUp();
    $this->installEntitySchema('node');
    $this->installEntitySchema('user');
    $this->installConfig(['system', 'node']);

    // Créer le type de contenu pour les tests
    NodeType::create(['type' => 'article', 'name' => 'Article'])->save();
  }

  public function testLayoutBuilderActivation(): void {
    $storage = \Drupal::entityTypeManager()->getStorage('entity_view_display');

    // Créer le display d'abord
    $display = $storage->create([
      'targetEntityType' => 'node',
      'bundle'           => 'article',
      'mode'             => 'default',
      'status'           => TRUE,
    ]);

    // Activer Layout Builder
    $display->enableLayoutBuilder()->setOverridable()->save();

    // Recharger depuis le storage pour vérifier la persistance
    $display = $storage->load('node.article.default');
    $this->assertTrue($display->isLayoutBuilderEnabled());
    $this->assertTrue($display->isOverridable());
  }

  public function testAddSectionProgrammatically(): void {
    use Drupal\layout_builder\Section;

    $display = \Drupal::entityTypeManager()
      ->getStorage('entity_view_display')
      ->load('node.article.default');

    $display->enableLayoutBuilder()->save();

    $section = new Section('layout_onecol');
    $display->appendSection($section);
    $display->save();

    $display = \Drupal::entityTypeManager()
      ->getStorage('entity_view_display')
      ->load('node.article.default');

    $this->assertCount(1, $display->getSections());
  }
}
```

---

## 9. Anti-patterns Layout Builder

| ❌ | ✅ | Raison |
|----|----|--------|
| Modifier les données de sections directement en base SQL | `$display->appendSection()` + `$display->save()` | Intégrité du système de config + cache |
| `$display->isOverridable()` sans vérifier `isLayoutBuilderEnabled()` | Toujours vérifier `isLayoutBuilderEnabled()` d'abord | Exception si LB inactif |
| Modifier une section sans invalider le cache | `Cache::invalidateTags(['node:' . $nid])` après chaque modification | Layout mis en cache agressivement |
| Utiliser `layout_twocol` (deprecated) | `layout_twocol_section` | Supprimé en D10+ |
| Charger le display sans vérifier s'il existe | Vérifier `if ($display)` avant `->enableLayoutBuilder()` | Exception sur display inexistant |
| `SectionComponent` avec un UUID hardcodé | `\Drupal::service('uuid')->generate()` | Collision d'UUID en cas de déploiement multi-env |
| Oublier `#[EventSubscriber]` attribute (D11+) | Ajouter le tag `event_subscriber` dans services.yml **ou** l'attribute PHP | Le subscriber ne s'enregistre pas |

---

## 10. Commandes Drush utiles

```bash
# Vérifier quels displays ont Layout Builder actif
docker compose exec php drush php:eval "
  \$displays = \Drupal::entityTypeManager()
    ->getStorage('entity_view_display')
    ->loadMultiple();
  foreach (\$displays as \$id => \$display) {
    if (method_exists(\$display, 'isLayoutBuilderEnabled') && \$display->isLayoutBuilderEnabled()) {
      echo \$id . ' (overridable: ' . (\$display->isOverridable() ? 'yes' : 'no') . ')' . PHP_EOL;
    }
  }
"

# Vider le cache Layout Builder spécifiquement
docker compose exec php drush cache:rebuild
```
