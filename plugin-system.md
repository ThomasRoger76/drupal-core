# Plugin System

## Pourquoi les Plugins ?

Les plugins permettent de définir des comportements interchangeables que Drupal découvre automatiquement via le namespace PSR-4. Sans modifier le core, tu peux créer des blocs, formatters, widgets, conditions, actions, etc.

## Annotation (D8-D10) vs Attribute PHP 8.2+ (D11+)

```php
// AVANT — Annotation (dépréciée en D11, encore fonctionnelle)
/**
 * @Block(
 *   id = "articles_recents",
 *   admin_label = @Translation("Articles récents"),
 *   category = @Translation("Contenu"),
 * )
 */

// APRÈS — PHP 8 Attribute (standard D11+, optionnel D10)
#[Block(
  id: 'articles_recents',
  admin_label: new TranslatableMarkup('Articles récents'),
  category: new TranslatableMarkup('Contenu'),
)]
```

**Règle :** nouveau code sur D10+ → utiliser les Attributes. Code legacy D8/D9 → Annotations encore valides.

---

## Types de Plugins et Namespaces

| Type | Namespace | Classe de base |
|------|-----------|----------------|
| Block | `Plugin\Block\` | `BlockBase` |
| FieldFormatter | `Plugin\Field\FieldFormatter\` | `FormatterBase` |
| FieldWidget | `Plugin\Field\FieldWidget\` | `WidgetBase` |
| Condition | `Plugin\Condition\` | `ConditionPluginBase` |
| Action | `Plugin\Action\` | `ActionBase` |
| Filter | `Plugin\Filter\` | `FilterBase` |
| QueueWorker | `Plugin\QueueWorker\` | `QueueWorkerBase` |

---

## Exemple Complet — Block Plugin

```php
<?php
// src/Plugin/Block/ArticlesRecentBlock.php
namespace Drupal\mon_module\Plugin\Block;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Access\AccessResultInterface;
use Drupal\Core\Block\Attribute\Block;
use Drupal\Core\Block\BlockBase;
use Drupal\Core\Cache\Cache;
use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Drupal\Core\Session\AccountInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Symfony\Component\DependencyInjection\ContainerInterface;

#[Block(
  id: 'articles_recents',
  admin_label: new TranslatableMarkup('Articles récents'),
)]
// ContainerFactoryPluginInterface : obligatoire pour les plugins qui ont besoin de DI.
// Permet à Drupal d'appeler create() sur le plugin au lieu du constructeur standard.
// Sans cette interface, Drupal instancie le plugin via __construct() à 3 arguments seulement.
final class ArticlesRecentBlock extends BlockBase implements ContainerFactoryPluginInterface {

  public function __construct(
    array $configuration,
    string $plugin_id,
    mixed $plugin_definition,
    private readonly EntityTypeManagerInterface $entityTypeManager,
  ) {
    parent::__construct($configuration, $plugin_id, $plugin_definition);
  }

  public static function create(
    ContainerInterface $container,
    array $configuration,
    string $plugin_id,
    mixed $plugin_definition,
  ): static {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
      $container->get('entity_type.manager'),
    );
  }

  // OBLIGATOIRE si le bloc a une config — définit les valeurs par défaut
  public function defaultConfiguration(): array {
    return ['max_items' => 5] + parent::defaultConfiguration();
  }

  // Surcharger pour contrôler l'accès au bloc depuis le code (pas l'UI)
  // IMPORTANT : protected, pas public — signature imposée par BlockBase
  protected function blockAccess(AccountInterface $account): AccessResultInterface {
    return AccessResult::allowedIfHasPermission($account, 'access content');
  }

  // Surcharger pour des cache tags/contexts propres au bloc (complète #cache dans build())
  public function getCacheTags(): array {
    return Cache::mergeTags(parent::getCacheTags(), ['node_list:article']);
  }

  public function getCacheContexts(): array {
    return Cache::mergeContexts(parent::getCacheContexts(), ['user.permissions']);
  }

  // Formulaire de configuration du bloc (affiché dans l'UI Block)
  public function blockForm(array $form, FormStateInterface $form_state): array {
    $form = parent::blockForm($form, $form_state);
    $form['max_items'] = [
      '#type'          => 'number',
      '#title'         => $this->t('Nombre d\'articles'),
      '#default_value' => $this->configuration['max_items'],  // garanti par defaultConfiguration()
      '#min'           => 1,
      '#max'           => 20,
    ];
    return $form;
  }

  public function blockSubmit(array $form, FormStateInterface $form_state): void {
    $this->configuration['max_items'] = $form_state->getValue('max_items');
  }

  public function build(): array {
    $max = $this->configuration['max_items'];  // Toujours défini via defaultConfiguration()
    $storage = $this->entityTypeManager->getStorage('node');

    $nids = $storage->getQuery()
      ->accessCheck(TRUE)
      ->condition('type', 'article')
      ->condition('status', 1)
      ->sort('created', 'DESC')
      ->range(0, $max)
      ->execute();

    if (empty($nids)) {
      return ['#markup' => $this->t('Aucun article.')];
    }

    $nodes = $storage->loadMultiple($nids);
    $items = array_map(fn($n) => $n->toLink()->toString(), $nodes);

    // Pas de #cache ici — géré par getCacheTags()/getCacheContexts() définis sur la classe
    return [
      '#theme' => 'item_list',
      '#items' => $items,
    ];
  }
}
```

---

## Plugin Discovery — Comment ça marche

Drupal scanne automatiquement les classes dans `src/Plugin/` grâce au mapping PSR-4 du `.info.yml` :

```yaml
# mon_module.info.yml
name: Mon Module
type: module
core_version_requirement: ^10 || ^11
```

Le namespace `Drupal\mon_module\Plugin\Block\` est découvert via l'autoloader Composer.
**Drupal ne scan pas d'autres dossiers** — les plugins DOIVENT être dans `src/Plugin/<Type>/`.

---

## Lire le Core comme référence canonique

Avant d'implémenter un type de plugin, lire la classe de base dans le Core :

```bash
# Trouver la classe de base d'un plugin
find web/core -name "BlockBase.php" -path "*/Plugin/*"
# → web/core/lib/Drupal/Core/Block/BlockBase.php

# Voir les méthodes disponibles à surcharger
grep -n "public function" web/core/lib/Drupal/Core/Block/BlockBase.php
```

Les modules `block_content`, `node`, `system` fournissent de bons exemples concrets.

---

## Exemple Minimal — FieldFormatter

```php
<?php
// src/Plugin/Field/FieldFormatter/MonFormatter.php
namespace Drupal\mon_module\Plugin\Field\FieldFormatter;

use Drupal\Core\Field\Attribute\FieldFormatter;
use Drupal\Core\Field\FieldItemListInterface;
use Drupal\Core\Field\FormatterBase;
use Drupal\Core\StringTranslation\TranslatableMarkup;

#[FieldFormatter(
  id: 'mon_module_custom',
  label: new TranslatableMarkup('Mon formatter'),
  field_types: ['string', 'text'],          // Types de champs compatibles
)]
final class MonFormatter extends FormatterBase {

  public function viewElements(FieldItemListInterface $items, $langcode): array {
    $elements = [];
    foreach ($items as $delta => $item) {
      $elements[$delta] = [
        '#markup' => '<strong>' . htmlspecialchars($item->value) . '</strong>',
      ];
    }
    return $elements;
  }
}
```

Le FieldFormatter apparaît dans "Manage display" du Content Type dès que le module est activé.
