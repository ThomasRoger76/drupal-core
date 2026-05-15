# Multilingual Drupal — Language, Content Translation, Config Translation

## Vue d'ensemble

Drupal gère le multilingue via quatre modules core : `language` (gestion des langues), `content_translation` (traduction des entités), `config_translation` (traduction de la configuration), et `locale` (traduction des chaînes d'interface). Ce guide couvre l'API PHP, Twig, les routes et les anti-patterns.

```bash
# Activer les modules multilingues essentiels
docker compose exec php drush en language content_translation config_translation locale -y
docker compose exec php drush cr
```

---

## 1. Services multilingues essentiels

```php
use Drupal\Core\Language\LanguageManagerInterface;
use Drupal\Core\Language\LanguageInterface;

// src/Service/MonService.php
class MonService {

  public function __construct(
    private readonly LanguageManagerInterface $languageManager,
  ) {}

  /**
   * Retourne le code ISO de la langue courante (ex: 'fr', 'en', 'ar').
   */
  public function getLangueCourante(): string {
    return $this->languageManager->getCurrentLanguage()->getId();
  }

  /**
   * Retourne toutes les langues actives du site.
   *
   * @return \Drupal\Core\Language\LanguageInterface[]
   */
  public function getToutesLesLangues(): array {
    return $this->languageManager->getLanguages();
    // Retourne : ['fr' => LanguageObject, 'en' => LanguageObject, ...]
  }

  /**
   * Retourne la langue par défaut du site.
   */
  public function getLangueParDefaut(): string {
    return $this->languageManager->getDefaultLanguage()->getId();
  }

  /**
   * Vérifie si la langue courante est RTL (arabe, hébreu, persan...).
   */
  public function estRTL(): bool {
    return $this->languageManager->getCurrentLanguage()->getDirection()
      === LanguageInterface::DIRECTION_RTL;
  }

  /**
   * Retourne les informations d'une langue spécifique.
   */
  public function getLangue(string $langcode): ?LanguageInterface {
    return $this->languageManager->getLanguage($langcode);
  }
}
```

```yaml
# mon_module.services.yml
services:
  mon_module.service:
    class: Drupal\mon_module\Service\MonService
    arguments:
      - '@language_manager'
```

---

## 2. Charger un nœud dans une langue spécifique

```php
use Drupal\node\Entity\Node;
use Drupal\node\NodeInterface;

/**
 * Charger la traduction française d'un nœud avec fallback.
 */
public function getNodeTraduit(int $nid): NodeInterface {
  $node = Node::load($nid);

  if (!$node) {
    throw new \InvalidArgumentException("Nœud $nid introuvable.");
  }

  $langcode = $this->languageManager->getCurrentLanguage()->getId();

  // Charger la traduction si disponible, sinon garder la langue originale
  if ($node->hasTranslation($langcode)) {
    return $node->getTranslation($langcode);
  }

  // Fallback explicite sur la langue source
  return $node;
}

/**
 * Charger via EntityTypeManager (recommandé dans les services injectés).
 */
public function getNodeAvecEntityManager(int $nid): NodeInterface {
  /** @var NodeInterface $node */
  $node = $this->entityTypeManager
    ->getStorage('node')
    ->load($nid);

  $langcode = $this->languageManager->getCurrentLanguage()->getId();

  return $node->hasTranslation($langcode)
    ? $node->getTranslation($langcode)
    : $node;
}

/**
 * Obtenir le titre dans toutes les langues disponibles.
 *
 * @return array<string, string>  ['fr' => 'Titre FR', 'en' => 'Title EN']
 */
public function getTitresMultilingues(NodeInterface $node): array {
  $titres = [];
  foreach ($node->getTranslationLanguages() as $langcode => $language) {
    $titres[$langcode] = $node->getTranslation($langcode)->label();
  }
  return $titres;
}
```

---

## 3. EntityQuery multilingue

```php
/**
 * EntityQuery retourne les entités dans leur langue par défaut.
 * Pour filtrer sur une langue spécifique, ajouter une condition langcode.
 */
public function getArticlesTraduits(string $langcode = NULL): array {
  $langcode = $langcode ?? $this->languageManager->getCurrentLanguage()->getId();

  $ids = $this->entityTypeManager
    ->getStorage('node')
    ->getQuery()
    ->condition('type', 'article')
    ->condition('status', 1)
    ->condition('langcode', $langcode)  // Filtrer sur la langue
    ->accessCheck(TRUE)
    ->execute();

  if (empty($ids)) {
    return [];
  }

  $nodes = $this->entityTypeManager->getStorage('node')->loadMultiple($ids);

  // Retourner les traductions dans la langue demandée
  return array_map(
    fn($node) => $node->hasTranslation($langcode)
      ? $node->getTranslation($langcode)
      : $node,
    $nodes
  );
}

/**
 * EntityQuery avec condition OR sur plusieurs langues (fallback).
 */
public function getNodesMultiLangues(array $langcodes): array {
  $orGroup = $this->entityTypeManager
    ->getStorage('node')
    ->getQuery()
    ->orConditionGroup();

  foreach ($langcodes as $langcode) {
    $orGroup->condition('langcode', $langcode);
  }

  $ids = $this->entityTypeManager
    ->getStorage('node')
    ->getQuery()
    ->condition('type', 'article')
    ->condition('status', 1)
    ->condition($orGroup)
    ->accessCheck(TRUE)
    ->execute();

  return $this->entityTypeManager->getStorage('node')->loadMultiple($ids);
}
```

---

## 4. Créer une traduction programmatiquement

```php
use Drupal\node\Entity\Node;

/**
 * Créer ou mettre à jour la traduction française d'un nœud.
 */
public function creerTraduction(int $nid, array $donnees_fr): void {
  $node = Node::load($nid);

  if (!$node) {
    return;
  }

  if (!$node->hasTranslation('fr')) {
    // Créer la nouvelle traduction
    $translation = $node->addTranslation('fr', [
      'title'  => $donnees_fr['titre'],
      'body'   => [
        'value'  => $donnees_fr['corps'],
        'format' => 'basic_html',
      ],
      'status' => 1,
    ]);
    $translation->save();
  }
  else {
    // Mettre à jour la traduction existante
    $translation = $node->getTranslation('fr');
    $translation->set('title', $donnees_fr['titre']);
    $translation->set('body', [
      'value'  => $donnees_fr['corps'],
      'format' => 'basic_html',
    ]);
    $translation->save();
  }
}

/**
 * Supprimer la traduction d'un nœud.
 */
public function supprimerTraduction(int $nid, string $langcode): void {
  $node = Node::load($nid);

  if ($node && $node->hasTranslation($langcode)
      && $langcode !== $node->language()->getId()) {
    // Ne jamais supprimer la langue source
    $node->removeTranslation($langcode);
    $node->save();
  }
}

/**
 * Créer plusieurs traductions en batch.
 *
 * @param array<string, array{titre: string, corps: string}> $traductions
 */
public function creerTraductionsMultiples(NodeInterface $node, array $traductions): void {
  foreach ($traductions as $langcode => $data) {
    if ($node->language()->getId() === $langcode) {
      continue;  // Ne pas remplacer la langue source via addTranslation
    }

    if ($node->hasTranslation($langcode)) {
      $t = $node->getTranslation($langcode);
    }
    else {
      $t = $node->addTranslation($langcode);
    }

    $t->set('title', $data['titre']);
    $t->set('body', ['value' => $data['corps'], 'format' => 'basic_html']);
    $t->set('status', 1);
    $t->save();
  }
}
```

---

## 5. Twig — affichage conditionnel par langue

```twig
{# Accéder à la langue courante dans Twig #}
{% set langue = language.id %}

{# Affichage conditionnel par langue #}
{% if langue == 'fr' %}
  <p>Contenu spécifique au français</p>
{% elseif langue == 'en' %}
  <p>English-specific content</p>
{% endif %}

{# Classe CSS selon la direction (RTL/LTR) #}
<div class="content {{ language.direction == 'rtl' ? 'rtl' : 'ltr' }}">
  {{ content }}
</div>

{# Générer un URL vers un nœud dans une langue spécifique #}
{% set url_fr = path('entity.node.canonical', {node: node.id}, {language: languages.fr}) %}
{% set url_en = path('entity.node.canonical', {node: node.id}, {language: languages.en}) %}

{# Switcher de langue — liens vers les traductions #}
{% if language_links is not empty %}
  <nav class="language-switcher" aria-label="Sélecteur de langue">
    {% for lang in language_links %}
      <a href="{{ lang.url }}"
         hreflang="{{ lang.langcode }}"
         {% if lang.langcode == language.id %}aria-current="page"{% endif %}>
        {{ lang.name }}
      </a>
    {% endfor %}
  </nav>
{% endif %}

{# Afficher le champ dans la langue traduite du nœud #}
{% if node.hasTranslation(language.id) %}
  {% set node_traduit = node.getTranslation(language.id) %}
  <h1>{{ node_traduit.label() }}</h1>
{% else %}
  <h1>{{ node.label() }}</h1>
{% endif %}
```

---

## 6. Config Translation — override par langue

```php
use Drupal\Core\Language\LanguageManagerInterface;

/**
 * Lire et écrire la configuration dans une langue spécifique.
 */
class ConfigTraductionService {

  public function __construct(
    private readonly LanguageManagerInterface $languageManager,
  ) {}

  /**
   * Lire la config dans une langue avec fallback sur la config par défaut.
   */
  public function getSiteNameTraduit(string $langcode = 'fr'): string {
    $override = $this->languageManager
      ->getLanguageConfigOverride($langcode, 'system.site');

    // getLanguageConfigOverride retourne NULL pour les clés non définies
    return $override->get('name')
      ?? \Drupal::config('system.site')->get('name');
  }

  /**
   * Modifier la configuration d'un module dans une langue.
   */
  public function setSujetEmailFr(string $sujet): void {
    $this->languageManager
      ->getLanguageConfigOverride('fr', 'mon_module.settings')
      ->set('email_subject', $sujet)
      ->save();
  }

  /**
   * Obtenir les overrides de toutes les langues pour une config.
   *
   * @return array<string, mixed>  ['fr' => ['name' => '...'], 'en' => [...]]
   */
  public function getAllOverrides(string $config_name): array {
    $result = [];
    foreach ($this->languageManager->getLanguages() as $langcode => $language) {
      $override = $this->languageManager->getLanguageConfigOverride($langcode, $config_name);
      $data = $override->get();
      if (!empty($data)) {
        $result[$langcode] = $data;
      }
    }
    return $result;
  }

  /**
   * Supprimer un override de config pour une langue.
   */
  public function supprimerOverride(string $langcode, string $config_name): void {
    $this->languageManager
      ->getLanguageConfigOverride($langcode, $config_name)
      ->delete();
  }
}
```

---

## 7. Routes multilingues — URL par langue

```yaml
# mon_module.routing.yml
# Drupal génère automatiquement les URLs avec préfixe de langue
# ex: /fr/ma-page, /en/my-page si language_url est configuré

mon_module.page:
  path: '/ma-page'
  defaults:
    _controller: '\Drupal\mon_module\Controller\MonController::page'
    _title: 'Ma Page'
  requirements:
    _permission: 'access content'
  # Les URLs traduites sont gérées automatiquement par le module 'language'
  # via /admin/config/regional/language/detection/url
```

```php
// Générer une URL dans une langue spécifique via PHP
use Drupal\Core\Url;
use Drupal\Core\Language\LanguageInterface;

public function getUrlTraduite(string $route, array $params, string $langcode): string {
  $langue = $this->languageManager->getLanguage($langcode);

  return Url::fromRoute($route, $params, [
    'language' => $langue,
    'absolute' => TRUE,
  ])->toString();
}

// Obtenir les liens de langue pour un nœud (pour le bloc switcher)
public function getLiensLangue(NodeInterface $node): array {
  $liens = [];
  foreach ($node->getTranslationLanguages() as $langcode => $language) {
    $translation = $node->getTranslation($langcode);
    $liens[$langcode] = [
      'url'      => $translation->toUrl('canonical', ['language' => $language])->toString(),
      'name'     => $language->getName(),
      'langcode' => $langcode,
      'current'  => $langcode === $this->languageManager->getCurrentLanguage()->getId(),
    ];
  }
  return $liens;
}
```

---

## 8. hook_language_switch_links_alter — personnaliser le switcher

```php
use Drupal\Core\Language\LanguageInterface;

/**
 * Implements hook_language_switch_links_alter().
 *
 * Modifier les liens du bloc switcher de langue.
 */
function mon_module_language_switch_links_alter(
  array &$links,
  string $type,
  \Drupal\Core\Url $url
): void {
  foreach ($links as $langcode => &$link) {
    // Ajouter le code de langue comme attribut data
    $link['attributes']['data-langcode'] = $langcode;

    // Ajouter hreflang pour le SEO
    $link['attributes']['hreflang'] = $langcode;
  }
}
```

---

## 9. Traduction des chaînes dans les services PHP

```php
use Drupal\Core\StringTranslation\StringTranslationTrait;
use Drupal\Core\StringTranslation\TranslationInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;

class MonService {
  use StringTranslationTrait;

  public function __construct(
    private readonly TranslationInterface $stringTranslation,
  ) {}

  public function getMessage(): TranslatableMarkup {
    // $this->t() utilise StringTranslationTrait — testable par injection
    return $this->t('Bonjour @name, bienvenue !', ['@name' => 'Roger']);
  }

  public function getMessagesStatiques(): array {
    // new TranslatableMarkup() pour les chaînes hors service (ex: constantes)
    return [
      'titre'       => new TranslatableMarkup('Titre de la page'),
      'description' => new TranslatableMarkup('Description multilingue'),
    ];
  }
}
```

```yaml
# mon_module.services.yml — injecter string_translation
services:
  mon_module.service:
    class: Drupal\mon_module\Service\MonService
    arguments:
      - '@string_translation'
    calls:
      - [setStringTranslation, ['@string_translation']]
```

---

## 10. Anti-patterns multilingue

| ❌ | ✅ | Raison |
|----|----|--------|
| `$node->label()` sans vérifier la traduction | `$node->getTranslation($langcode)->label()` | Retourne la langue par défaut, pas la courante |
| EntityQuery sans `->condition('langcode', $langcode)` | Toujours filtrer sur la langue courante | Double entrées / résultats dans la mauvaise langue |
| `t('texte')` statique dans un service PHP | `$this->t()` via `StringTranslationTrait` | `t()` global non testable, non injectable |
| Hardcoder `'langcode' => 'fr'` | `$this->languageManager->getCurrentLanguage()->getId()` | Non portable, casse les autres langues |
| `$node->addTranslation()` sur la langue source | Vérifier `$langcode !== $node->language()->getId()` | Erreur "Cannot add translation in the source language" |
| Modifier `$node->set('title', ...)` sans cibler la traduction | `$node->getTranslation($langcode)->set('title', ...)` | Modifie la langue source, pas la traduction |
| Accéder à `languages.fr` en Twig sans vérifier son existence | `{% if languages.fr is defined %}` | Twig exception si la langue n'existe pas |
| `drupal_static_reset('language_list')` en runtime | Ne pas resetter les langues en dehors de tests | Race condition en multithread |
| Stocker le langcode en session plutôt que d'utiliser le LanguageManager | Injecter `LanguageManagerInterface` | Désynchronisation avec le système de détection de langue Drupal |

---

## 11. Commandes Drush utiles

```bash
# Lister les langues actives
docker compose exec php drush php:eval "
  foreach (\Drupal::languageManager()->getLanguages() as \$id => \$lang) {
    echo \$id . ' : ' . \$lang->getName() . ' (' . \$lang->getDirection() . ')' . PHP_EOL;
  }
"

# Ajouter une langue via Drush
docker compose exec php drush language-add fr

# Importer les traductions d'interface pour une langue
docker compose exec php drush locale-update --langcodes=fr

# Vérifier les chaînes non traduites
docker compose exec php drush locale-check
```
