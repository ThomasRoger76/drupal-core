# Forms API

## Trois Classes de Base

| Classe | Usage | Quand |
|--------|-------|-------|
| `FormBase` | Formulaire standalone | Actions, recherches, formulaires custom |
| `ConfigFormBase` | Settings admin | Paramètres exportables en config YAML |
| `ConfirmFormBase` | Confirmation avant action destructive | Suppression, reset |

---

## FormBase — Pattern Complet avec DI

`FormBase` implémente `ContainerInjectionInterface` — toujours implémenter `create()` si un service est nécessaire :

```php
<?php
// src/Form/MonForm.php
namespace Drupal\mon_module\Form;

use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Form\FormBase;
use Drupal\Core\Form\FormStateInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

final class MonForm extends FormBase {

  public function __construct(
    private readonly EntityTypeManagerInterface $entityTypeManager,
  ) {}

  public static function create(ContainerInterface $container): static {
    return new static($container->get('entity_type.manager'));
  }

  public function getFormId(): string {
    return 'mon_module_mon_form';   // Convention : module_description_form
  }

  public function buildForm(array $form, FormStateInterface $form_state): array {
    $types = $this->entityTypeManager->getStorage('node_type')->loadMultiple();

    $form['type'] = [
      '#type'     => 'select',
      '#title'    => $this->t('Type de contenu'),
      '#options'  => array_map(fn($t) => $t->label(), $types),
      '#required' => TRUE,
    ];

    $form['titre'] = [
      '#type'      => 'textfield',
      '#title'     => $this->t('Titre'),
      '#required'  => TRUE,
      '#maxlength' => 255,
    ];

    $form['actions'] = ['#type' => 'actions'];
    $form['actions']['submit'] = [
      '#type'  => 'submit',
      '#value' => $this->t('Envoyer'),
    ];

    return $form;
  }

  public function validateForm(array &$form, FormStateInterface $form_state): void {
    if (strlen($form_state->getValue('titre')) < 3) {
      $form_state->setErrorByName('titre', $this->t('Le titre doit faire au moins 3 caractères.'));
    }
  }

  public function submitForm(array &$form, FormStateInterface $form_state): void {
    $this->messenger()->addStatus($this->t('Formulaire soumis avec succès.'));
    $form_state->setRedirect('mon_module.liste');
  }
}
```

---

## ConfigFormBase — Settings Admin

```php
<?php
namespace Drupal\mon_module\Form;

use Drupal\Core\Form\ConfigFormBase;
use Drupal\Core\Form\FormStateInterface;

final class SettingsForm extends ConfigFormBase {

  protected function getEditableConfigNames(): array {
    return ['mon_module.settings'];
  }

  public function getFormId(): string {
    return 'mon_module_settings';
  }

  public function buildForm(array $form, FormStateInterface $form_state): array {
    $config = $this->config('mon_module.settings');
    $form['max_items'] = [
      '#type'          => 'number',
      '#title'         => $this->t('Nombre maximum d\'items'),
      '#default_value' => $config->get('max_items') ?? 10,
    ];
    return parent::buildForm($form, $form_state);  // Ajoute le bouton Save
  }

  public function submitForm(array &$form, FormStateInterface $form_state): void {
    $this->config('mon_module.settings')
      ->set('max_items', $form_state->getValue('max_items'))
      ->save();
    parent::submitForm($form, $form_state);  // Message "Configuration saved"
  }
}
```

---

## AJAX API — Pattern Complet

```php
public function buildForm(array $form, FormStateInterface $form_state): array {

  $form['categorie'] = [
    '#type'    => 'select',
    '#title'   => $this->t('Catégorie'),
    '#options' => $this->getCategories(),
    '#ajax'    => [
      'callback' => '::updateSousCategories',
      'wrapper'  => 'sous-categories-wrapper',  // ID du conteneur à remplacer
      'effect'   => 'fade',
      'progress' => ['type' => 'throbber'],
    ],
  ];

  // Wrapper ciblé par AJAX — DOIT entourer tout ce qui sera mis à jour
  $form['sous_cat_container'] = [
    '#type'       => 'container',
    '#attributes' => ['id' => 'sous-categories-wrapper'],
  ];
  $form['sous_cat_container']['sous_categorie'] = [
    '#type'    => 'select',
    '#title'   => $this->t('Sous-catégorie'),
    '#options' => $this->getSousCategories($form_state->getValue('categorie') ?? ''),
  ];

  $form['submit'] = ['#type' => 'submit', '#value' => $this->t('Valider')];

  return $form;
}

// Callback AJAX — retourne uniquement le sous-formulaire à remplacer
public function updateSousCategories(array &$form, FormStateInterface $form_state): array {
  // Identifier l'élément déclencheur si plusieurs éléments partagent la même callback
  $triggering = $form_state->getTriggeringElement();
  // $triggering['#name'] contient le nom de l'élément qui a déclenché l'AJAX

  return $form['sous_cat_container'];
}
```

**Forcer la reconstruction du form (multi-étapes ou AJAX complexe) :**
```php
public function submitForm(array &$form, FormStateInterface $form_state): void {
  $form_state->setRebuild(TRUE);                // Reconstruire au lieu de rediriger
  $form_state->set('step', ($form_state->get('step') ?? 1) + 1);  // Stocker l'état
}
```

**Erreurs fréquentes :**
- `wrapper` ne correspond pas à l'`id` → rien ne se met à jour
- Callback retourne tout `$form` au lieu du fragment → duplication
- Oublier `'#type' => 'container'` sur le wrapper → id non rendu dans le HTML

---

## `#states` API — Afficher/Cacher sans AJAX

Afficher ou masquer des éléments en JavaScript pur, sans aller-retour serveur. À utiliser dans `buildForm()` :

```php
public function buildForm(array $form, FormStateInterface $form_state): array {

  $form['type'] = [
    '#type'    => 'select',
    '#title'   => $this->t('Type'),
    '#options' => ['standard' => 'Standard', 'custom' => 'Personnalisé'],
  ];

  // Visible et required uniquement si type = 'custom'
  $form['custom_value'] = [
    '#type'   => 'textfield',
    '#title'  => $this->t('Valeur personnalisée'),
    '#states' => [
      'visible'  => [':input[name="type"]' => ['value' => 'custom']],
      'required' => [':input[name="type"]' => ['value' => 'custom']],
    ],
  ];

  // Condition ET (les deux doivent être vrais)
  $form['extra'] = [
    '#type'   => 'textfield',
    '#title'  => $this->t('Extra (si enabled ET custom)'),
    '#states' => [
      'visible' => [
        ':input[name="enabled"]' => ['checked' => TRUE],
        ':input[name="type"]'    => ['value' => 'custom'],
      ],
    ],
  ];

  // Condition OU (tableau de tableaux)
  $form['either'] = [
    '#type'   => 'textfield',
    '#title'  => $this->t('Visible si type = a OU b'),
    '#states' => [
      'visible' => [
        [':input[name="type"]' => ['value' => 'a']],
        [':input[name="type"]' => ['value' => 'b']],
      ],
    ],
  ];

  return $form;
}
```

**États disponibles :** `visible`, `invisible`, `enabled`, `disabled`, `required`, `optional`, `checked`, `unchecked`

**`#states` vs AJAX :** utiliser `#states` pour le pur affichage/masquage. AJAX est réservé aux cas où les options elles-mêmes doivent être rechargées depuis le serveur.

---

## hook_form_alter — Modifier un Formulaire Existant

```php
// DANS mon_module.module — ou src/Hook/ avec #[Hook('form_node_article_form_alter')] sur D11

// Cibler UN formulaire précis — PRÉFÉRABLE
function mon_module_form_node_article_form_alter(array &$form, FormStateInterface $form_state): void {
  $form['field_date']['#required'] = TRUE;
  $form['#validate'][] = 'mon_module_article_custom_validate';
}

function mon_module_article_custom_validate(array &$form, FormStateInterface $form_state): void {
  // Validation custom
}

// Modifier TOUS les formulaires (avec filtre)
function mon_module_form_alter(array &$form, FormStateInterface $form_state, string $form_id): void {
  if (in_array($form_id, ['node_article_form', 'node_article_edit_form'], TRUE)) {
    $form['field_date']['#required'] = TRUE;
  }
}
```

---

## Types d'Éléments Courants

| `#type` | Usage |
|---------|-------|
| `textfield` | Champ texte court |
| `textarea` | Texte long |
| `select` | Liste déroulante |
| `checkboxes` | Cases à cocher multiples |
| `radios` | Boutons radio |
| `entity_autocomplete` | Recherche d'entité Drupal |
| `managed_file` | Upload de fichier |
| `date` | Sélecteur de date |
| `number` | Champ numérique |
| `password` | Mot de passe masqué |
| `email` | Champ email (validation HTML5) |
| `url` | Champ URL (validation HTML5) |
| `fieldset` | Groupe non collapsible |
| `details` | Groupe collapsible |
| `container` | Wrapper sans rendu visuel (AJAX) |
| `hidden` | Champ caché |
| `weight` | Champ de poids (drag & drop) |
