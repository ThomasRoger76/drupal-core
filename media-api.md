# Media API — Module Core Drupal (D8.5+)

## Vue d'ensemble

Le module Media (core depuis D8.5) gère les assets réutilisables : images, documents, vidéos, audios, vidéos distantes. Il découple les fichiers physiques des entités qui les référencent via des **Media Source plugins**.

```bash
docker compose exec php drush en media media_library -y
```

---

## Types Media (bundles) par défaut

| Bundle | Source Plugin | Champ source |
|--------|--------------|--------------|
| `image` | `image` | `field_media_image` |
| `document` | `file` | `field_media_document` |
| `video` | `video_file` | `field_media_video_file` |
| `audio` | `audio_file` | `field_media_audio_file` |
| `remote_video` | `oembed:video` | `field_media_oembed_video` |

Configuration UI : `/admin/structure/media`

---

## Créer un Media programmatiquement

### Image (avec création de fichier)

```php
use Drupal\media\Entity\Media;
use Drupal\file\Entity\File;

// Étape 1 : créer l'entité File (le fichier physique doit déjà exister)
$file = File::create([
  'uri' => 'public://uploads/mon-image.jpg',
  'status' => 1,
  'uid' => \Drupal::currentUser()->id(),
]);
$file->save();

// Étape 2 : créer le Media de type image
$media = Media::create([
  'bundle' => 'image',
  'name' => 'Mon image produit',
  'field_media_image' => [
    'target_id' => $file->id(),
    'alt' => 'Description alternative pour l\'accessibilité',
    'title' => 'Titre au survol',
    'width' => 1200,
    'height' => 800,
  ],
  'status' => 1,  // 1 = publié, 0 = dépublié
  'uid' => \Drupal::currentUser()->id(),
]);
$media->save();

$media_id = $media->id();
```

### Document PDF

```php
$file = File::create([
  'uri' => 'public://documents/rapport-annuel.pdf',
  'status' => 1,
]);
$file->save();

$media = Media::create([
  'bundle' => 'document',
  'name' => 'Rapport annuel 2024',
  'field_media_document' => ['target_id' => $file->id()],
  'status' => 1,
]);
$media->save();
```

### Copier un fichier vers le système de fichiers Drupal

```php
use Drupal\Core\File\FileSystemInterface;

$file_system = \Drupal::service('file_system');
$file_repository = \Drupal::service('file.repository');

// Copier depuis une URL externe
$data = file_get_contents('https://exemple.com/image.jpg');
$file = $file_repository->writeData(
  $data,
  'public://uploads/image-' . time() . '.jpg',
  FileSystemInterface::EXISTS_RENAME  // ou EXISTS_REPLACE, EXISTS_ERROR
);
```

---

## Charger et afficher un Media depuis PHP

### Charger par ID

```php
use Drupal\media\Entity\Media;

$media = Media::load($mid);
if (!$media) {
  // Media inexistant ou accès refusé
  return;
}

// Vérifier l'accès
if (!$media->access('view')) {
  return;
}
```

### Obtenir l'URL du fichier source

```php
// Pour un Media de type image
/** @var \Drupal\file\Entity\File $file */
$file = $media->field_media_image->entity;
$file_uri = $file->getFileUri(); // ex: public://uploads/mon-image.jpg

// Générer l'URL absolue
$file_url_generator = \Drupal::service('file_url_generator');
$url = $file_url_generator->generateAbsoluteString($file_uri);
// Résultat : https://monsite.com/sites/default/files/uploads/mon-image.jpg

// URL relative (pour les render arrays)
$relative_url = $file_url_generator->generateString($file_uri);
```

### Obtenir l'URL d'un style d'image

```php
use Drupal\image\Entity\ImageStyle;

$image_style = ImageStyle::load('medium'); // ou 'thumbnail', 'large', etc.
$styled_url = $image_style->buildUrl($file_uri);
```

### Render array pour un Media

```php
// Vue par défaut du Media
$build = \Drupal::entityTypeManager()
  ->getViewBuilder('media')
  ->view($media, 'default');

// Vue spécifique (view mode personnalisé)
$build = \Drupal::entityTypeManager()
  ->getViewBuilder('media')
  ->view($media, 'thumbnail');

// Avec cache correctement configuré
$build['#cache']['tags'] = $media->getCacheTags();
$build['#cache']['contexts'] = ['user.permissions'];
```

---

## Service `file.usage` — Tracker les fichiers référencés

```php
$file_usage = \Drupal::service('file.usage');

// Enregistrer qu'un module utilise un fichier
$file_usage->add($file, 'mon_module', 'node', $node->id());

// Récupérer les usages d'un fichier
$usages = $file_usage->listUsage($file);
// Résultat : ['node' => ['1' => 2, '5' => 1], 'media' => ['3' => 1]]

// Supprimer un usage
$file_usage->delete($file, 'mon_module', 'node', $node->id());

// Un fichier sans usage est éligible à la suppression automatique
// (si "Delete orphaned files" est activé dans la config)
```

---

## Media Source Plugins

Chaque type Media est associé à un **Media Source plugin** qui détermine comment extraire les métadonnées du fichier source.

### Sources disponibles en core

| Plugin ID | Bundle | Fonctionnalité |
|-----------|--------|---------------|
| `image` | image | Dimensions, alt, title |
| `file` | document | Mime-type, taille |
| `video_file` | video | Durée (si FFmpeg) |
| `audio_file` | audio | Durée (si FFmpeg) |
| `oembed:video` | remote_video | YouTube/Vimeo via oEmbed |

### Créer un Media Source Plugin custom

```php
// src/Plugin/media/Source/ExternalApi.php
namespace Drupal\mon_module\Plugin\media\Source;

use Drupal\media\MediaSourceBase;
use Drupal\media\MediaInterface;

/**
 * Source plugin pour les assets d'une API externe.
 *
 * @MediaSource(
 *   id = "external_api",
 *   label = @Translation("API Externe"),
 *   description = @Translation("Assets hébergés sur une API externe"),
 *   allowed_field_types = {"string"},
 *   default_thumbnail_filename = "generic.png",
 * )
 */
class ExternalApi extends MediaSourceBase {

  /**
   * {@inheritdoc}
   */
  public function getMetadataAttributes(): array {
    return [
      'asset_id' => $this->t('ID de l\'asset'),
      'asset_title' => $this->t('Titre de l\'asset'),
      'asset_url' => $this->t('URL publique'),
      'thumbnail_url' => $this->t('URL de la miniature'),
    ];
  }

  /**
   * {@inheritdoc}
   */
  public function getMetadata(MediaInterface $media, string $attribute_name): mixed {
    $source_field = $this->getSourceFieldValue($media);
    if (empty($source_field)) {
      return NULL;
    }

    // Récupérer les métadonnées depuis l'API
    $data = $this->fetchAssetData($source_field);

    return match ($attribute_name) {
      'asset_id' => $data['id'] ?? NULL,
      'asset_title' => $data['title'] ?? NULL,
      'asset_url' => $data['url'] ?? NULL,
      'thumbnail_url' => $data['thumbnail'] ?? NULL,
      default => parent::getMetadata($media, $attribute_name),
    };
  }

  /**
   * Récupère les données depuis l'API externe.
   */
  protected function fetchAssetData(string $asset_id): array {
    try {
      $response = \Drupal::httpClient()->get(
        "https://api.exemple.com/assets/{$asset_id}"
      );
      return json_decode($response->getBody()->getContents(), TRUE) ?? [];
    }
    catch (\Exception $e) {
      return [];
    }
  }

}
```

---

## Requêtes EntityQuery sur les Media

```php
// Charger tous les Media image publiés
$mids = \Drupal::entityQuery('media')
  ->condition('bundle', 'image')
  ->condition('status', 1)
  ->accessCheck(TRUE)
  ->sort('created', 'DESC')
  ->range(0, 10)
  ->execute();

$medias = Media::loadMultiple($mids);

// Trouver les Media référençant un fichier spécifique
$mids = \Drupal::entityQuery('media')
  ->condition('field_media_image.target_id', $file_id)
  ->accessCheck(TRUE)
  ->execute();
```

---

## Supprimer un Media proprement

```php
// Supprimer le Media (le fichier reste si d'autres entités le référencent)
$media->delete();

// Supprimer Media + fichier source si non utilisé ailleurs
$file = $media->field_media_image->entity;
$media->delete();

$usages = \Drupal::service('file.usage')->listUsage($file);
if (empty($usages)) {
  $file->delete();
}
```

---

## Media Library dans un formulaire

```php
// Dans buildForm() — champ de sélection Media Library
$form['image'] = [
  '#type' => 'media_library_element',
  '#allowed_bundles' => ['image'],
  '#title' => $this->t('Image principale'),
  '#description' => $this->t('Sélectionner ou uploader une image.'),
  '#cardinality' => 1,
  '#default_value' => $existing_media_id ?? NULL,
];
```

---

## Anti-patterns Media

| ❌ À éviter | ✅ Bonne pratique | Raison |
|------------|------------------|--------|
| Stocker des URLs externes en `field_text` | Utiliser `remote_video` ou source plugin custom | Pas de gestion des métadonnées, pas réutilisable |
| Créer un `File` sans Media associé | Toujours créer un Media pour les assets éditoriaux | Les fichiers orphelins sont supprimés automatiquement |
| Utiliser Media pour les assets du thème (CSS, JS) | Assets thème dans `/libraries` | Media est pour le contenu éditorial, pas les assets techniques |
| `file_get_contents()` pour de gros fichiers | `file.repository` + streaming | Mémoire PHP saturée |
| Ignorer `file.usage` lors de la suppression | Toujours vérifier avant de supprimer un fichier | Rupture de Media encore référencés |
| `\Drupal::service()` dans un constructeur de plugin | Injection via `create()` / `ContainerInterface` | Testabilité |

---

## See Also

- `drupal-core` — Entity API, EntityQuery, services
- `drupal-theming` — Render arrays pour Media, image styles dans Twig
- `json-api.md` — Exposer des Media via JSON:API
