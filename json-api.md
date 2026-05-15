# JSON:API — Module Core Drupal (D8.7+)

## Activation et configuration de base

```bash
docker compose exec php drush en jsonapi -y
docker compose exec php drush cr
```

Le module JSON:API est inclus dans le core depuis Drupal 8.7. Aucune dépendance contrib requise pour les cas d'usage standard.

---

## Endpoints auto-générés

Pattern : `/jsonapi/{entity_type}/{bundle}`

| Méthode | URL | Action |
|---------|-----|--------|
| GET | `/jsonapi/node/article` | Liste paginée des articles |
| GET | `/jsonapi/node/article/{uuid}` | Un article par UUID |
| POST | `/jsonapi/node/article` | Créer un article |
| PATCH | `/jsonapi/node/article/{uuid}` | Modifier un article |
| DELETE | `/jsonapi/node/article/{uuid}` | Supprimer un article |
| GET | `/jsonapi` | Index de tous les endpoints disponibles |

### Paramètres de requête

```bash
# Filtrer
GET /jsonapi/node/article?filter[title][value]=Mon titre
GET /jsonapi/node/article?filter[status][value]=1
GET /jsonapi/node/article?filter[field_tags.name][value]=Drupal

# Inclure des relations (évite le N+1)
GET /jsonapi/node/article?include=field_image,field_tags,uid

# Pagination
GET /jsonapi/node/article?page[offset]=0&page[limit]=10

# Tri (- pour DESC)
GET /jsonapi/node/article?sort=-created,title

# Sparse fieldsets — réduire la taille de la réponse
GET /jsonapi/node/article?fields[node--article]=title,body,created
```

---

## Structure d'une réponse JSON:API

```json
{
  "data": [
    {
      "type": "node--article",
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "attributes": {
        "title": "Mon article",
        "body": { "value": "<p>Contenu</p>", "format": "full_html" },
        "created": "2024-01-15T10:30:00+00:00",
        "status": true
      },
      "relationships": {
        "field_tags": {
          "data": [{ "type": "taxonomy_term--tags", "id": "uuid-du-terme" }]
        }
      }
    }
  ],
  "links": {
    "next": { "href": "/jsonapi/node/article?page[offset]=10&page[limit]=10" }
  },
  "meta": { "count": 42 }
}
```

---

## Access Control JSON:API

### Filtres autorisés par bundle

```php
// mon_module.module (D10) ou classe avec #[Hook] (D11)
use Drupal\Core\Access\AccessResult;
use Drupal\Core\Entity\EntityTypeInterface;
use Drupal\Core\Session\AccountInterface;
use Drupal\jsonapi\Access\EntityAccessChecker;

/**
 * Implements hook_jsonapi_ENTITY_TYPE_filter_access().
 */
function mon_module_jsonapi_node_filter_access(
  EntityTypeInterface $entity_type,
  AccountInterface $account
): array {
  return [
    JSONAPI_FILTER_AMONG_ALL => AccessResult::allowedIfHasPermission(
      $account,
      'access content'
    ),
    JSONAPI_FILTER_AMONG_PUBLISHED => AccessResult::allowedIfHasPermission(
      $account,
      'access content'
    ),
    JSONAPI_FILTER_AMONG_OWN => AccessResult::allowedIfHasPermission(
      $account,
      'access content'
    ),
  ];
}
```

### Contrôle d'accès par entité (standard)

JSON:API respecte les permissions Drupal natives. Si `node_access` refuse l'accès à un nœud, JSON:API le filtre automatiquement. Pas de configuration supplémentaire requise pour les cas standard.

---

## ResourceType — Personnaliser les champs exposés

### Via jsonapi_extras (module contrib)

```bash
composer require drupal/jsonapi_extras
docker compose exec php drush en jsonapi_extras -y
```

Interface admin : `/admin/config/services/jsonapi/resource_types`
- Renommer des champs dans la réponse (ex: `field_body` → `content`)
- Exclure des champs sensibles (ex: `field_password_hash`, `field_api_key`)
- Désactiver des resource types entiers

### Via hook (sans module contrib)

```php
use Drupal\jsonapi\ResourceType\ResourceType;

/**
 * Implements hook_jsonapi_resource_type_build_alter().
 */
function mon_module_jsonapi_resource_type_build_alter(ResourceType $resource_type): void {
  if ($resource_type->getEntityTypeId() === 'node' &&
      $resource_type->getBundle() === 'article') {
    // Désactiver un champ sensible
    $fields = $resource_type->getFields();
    if (isset($fields['field_internal_notes'])) {
      $fields['field_internal_notes']->setEnabled(FALSE);
    }
  }
}
```

---

## X-CSRF-Token pour les mutations (POST/PATCH/DELETE)

### Frontend JavaScript — même origine (cookie auth)

```javascript
// Étape 1 : obtenir le token CSRF
const tokenResponse = await fetch('/session/token');
const csrfToken = await tokenResponse.text();

// Étape 2 : créer un nœud
const response = await fetch('/jsonapi/node/article', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/vnd.api+json',
    'Accept': 'application/vnd.api+json',
    'X-CSRF-Token': csrfToken,
  },
  credentials: 'same-origin', // envoie les cookies de session
  body: JSON.stringify({
    data: {
      type: 'node--article',
      attributes: {
        title: 'Mon article créé via JSON:API',
        body: {
          value: '<p>Contenu de l\'article</p>',
          format: 'full_html',
        },
        status: true,
      },
      relationships: {
        field_tags: {
          data: [{ type: 'taxonomy_term--tags', id: 'uuid-du-terme' }],
        },
      },
    },
  }),
});

const created = await response.json();
console.log('Nœud créé :', created.data.id);
```

### Modifier un nœud existant (PATCH)

```javascript
// PATCH — uniquement les attributs à modifier
await fetch(`/jsonapi/node/article/${nodeUuid}`, {
  method: 'PATCH',
  headers: {
    'Content-Type': 'application/vnd.api+json',
    'X-CSRF-Token': csrfToken,
  },
  credentials: 'same-origin',
  body: JSON.stringify({
    data: {
      type: 'node--article',
      id: nodeUuid,
      attributes: { title: 'Titre modifié' },
    },
  }),
});
```

### Supprimer un nœud (DELETE)

```javascript
await fetch(`/jsonapi/node/article/${nodeUuid}`, {
  method: 'DELETE',
  headers: { 'X-CSRF-Token': csrfToken },
  credentials: 'same-origin',
});
```

---

## CORS pour frontend découplé

Dans `sites/default/services.yml` (ou `services.dev.yml` pour le dev uniquement) :

```yaml
parameters:
  cors.config:
    enabled: true
    # En-têtes autorisés dans les requêtes
    allowedHeaders:
      - 'Content-Type'
      - 'Authorization'
      - 'X-CSRF-Token'
      - 'Accept'
    # Origines autorisées (jamais '*' en production)
    allowedOrigins:
      - 'https://frontend.monsite.com'
      - 'https://app.monsite.com'
      - 'http://localhost:3000'    # Dev Next.js
      - 'http://localhost:5173'    # Dev Vite
    # Méthodes autorisées
    allowedMethods:
      - 'GET'
      - 'POST'
      - 'PATCH'
      - 'DELETE'
      - 'OPTIONS'
    # Exposer des en-têtes supplémentaires au frontend
    exposedHeaders: false
    # Autoriser les cookies/credentials cross-origin
    supportsCredentials: true
    maxAge: 1000
```

```bash
docker compose exec php drush cr
```

**Important :** `supportsCredentials: true` exige que `allowedOrigins` soit une liste explicite — jamais `['*']`.

---

## Authentification

| Méthode | Usage | Module |
|---------|-------|--------|
| Cookie session | Frontend même domaine | Core (aucun) |
| Basic Auth | API to API, dev/test uniquement | `basic_auth` (core) |
| OAuth2 / JWT | SPA découplée, mobile, production | `simple_oauth` (contrib) |

### Basic Auth (test uniquement)

```bash
# Ne jamais utiliser en production sans HTTPS
curl -u admin:password \
  -H 'Accept: application/vnd.api+json' \
  https://monsite.com/jsonapi/node/article
```

### Simple OAuth (production)

```bash
composer require drupal/simple_oauth
docker compose exec php drush en simple_oauth -y
# Configurer : /admin/config/people/simple_oauth
```

Flux OAuth2 : client credentials (machine-to-machine) ou authorization code (utilisateur).

---

## Anti-patterns JSON:API

| ❌ À éviter | ✅ Bonne pratique | Raison |
|------------|------------------|--------|
| Exposer `field_password_hash`, tokens, clés API | Exclure via `jsonapi_extras` ou hook | Fuite de données sensibles |
| Requêtes sans `?page[limit]` | Toujours paginer | OOM sur collections > 1000 items |
| `?include=` sur 5+ niveaux de profondeur | Limiter à 2 niveaux max | Requêtes SQL exponentielles |
| `?filter` sur champs non indexés | Ajouter un index DB ou utiliser Search API | Full scan = lenteur en production |
| CORS avec `allowedOrigins: ['*']` | Liste explicite d'origines | Sécurité cross-site |
| Basic Auth en production sans HTTPS | OAuth2 ou session cookie | Credentials en clair |
| Ignorer le code HTTP de la réponse | Vérifier 200/201/204 vs 4xx/5xx | JSON:API retourne des erreurs structurées |

---

## Gestion des erreurs JSON:API

```json
{
  "errors": [
    {
      "status": "422",
      "title": "Unprocessable Entity",
      "detail": "title: This value should not be null.",
      "source": { "pointer": "/data/attributes/title" }
    }
  ]
}
```

```javascript
if (!response.ok) {
  const error = await response.json();
  error.errors.forEach(e =>
    console.error(`[${e.source?.pointer}] ${e.detail}`)
  );
}
```

---

## See Also

- `drupal-core` — Entity API, permissions, accès
- `drupal-security` — CSRF, XSS, contrôle d'accès API
- `drupal-docker` — CORS en dev, services.yml local
