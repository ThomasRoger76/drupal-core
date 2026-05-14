# Changelog — drupal-core

Suivi des versions du skill. Incrémenter à chaque correction ou ajout significatif.

---

## v3.1 — 2026-05-14

**Bugs corrigés :**
- `blockAccess()` : `public` → `protected`, suppression `$return_as_object`, ajout `: AccessResultInterface`
- `php: ^8.2` → `php: '8.2'` dans module-structure.md (syntaxe Composer invalide dans .info.yml)
- `blockForm()` : suppression `?? 5` redondant avec `defaultConfiguration()`
- UPSERT : `time()` → `$this->time->getRequestTime()` (cohérence avec INSERT/UPDATE)

**Incohérences corrigées :**
- Double cache Block : `getCacheTags()`/`getCacheContexts()` ET `#cache` dans `build()` — un seul mécanisme maintenant
- Table services : `logger.factory` → distinction claire `logger.channel.mon_module` (recommandé) vs `logger.factory`
- `AliasManagerInterface` → `AliasRepositoryInterface` (deprecated D10+)
- `_custom_access` vs tag `access_check` : deux approches clairement séparées et expliquées
- `TimeInterface` : utilisé réellement dans `getRecentArticles()` (plus de dead code)

**Sémantique :**
- Anti-pattern "hooks dans des classes" : précision D8-D10 vs D11 `#[Hook]`
- `hook_form_alter` : commentaire "jamais dans une classe" → précision D11
- `use` statements ajoutés en tête du bloc hooks pour le fichier `.module`
- DI section database-api.md : `Connection` + `TimeInterface` + `LoggerInterface`

**Nouveau :**
- `lessons.md` : tracking des bugs trouvés en usage réel
- `CHANGELOG.md` : ce fichier

---

## v3.0 — 2026-05-14

**Absorbe drupal-backend (module-structure, database-api, services custom)**

**Ajouts :**
- `module-structure.md` : arborescence complète, `.permissions.yml` avec callbacks, `.links.*.yml`, config schema, workflow dev, troubleshooting
- `database-api.md` : SELECT/JOIN/GROUP BY/INSERT/UPDATE/DELETE/UPSERT/transactions, `hook_schema`, `hook_update_N` batch, query tags
- `#[Hook]` D11 avec DI complète
- `#states` API avec conditions ET/OU dans `buildForm()`
- DI dans `FormBase` avec `create()`
- `_title_callback`, `_entity_access`, `_form:` route shortcut
- `.libraries.yml`, `src/Hook/` dans arborescence
- `blockAccess()`, `getCacheTags()`, `getCacheContexts()` sur BlockBase
- `FieldFormatter` exemple minimal
- `Cache::invalidateTags()`
- Distinction `cache.default` / `cache.render` / `cache.data`

**Bugs corrigés v3.0 :**
- `ResponseEvent` import manquant dans EventSubscriber
- `\Drupal::currentUser()` → `$this->currentUser` dans Entity create
- `defaultConfiguration()` ajouté dans Block plugin
- `config_devel` clé erronée supprimée du `.info.yml`
- `drush event:debug` et `drush routes --route-name` remplacés
- `api_key` retiré du YAML + avertissement secrets
- `logger.factory` → `logger.channel.mon_module` avec `LoggerInterface`
- `time()` incohérences → `TimeInterface` dans classes

---

## v2.0 — 2026-05-14

**Bugs corrigés :**
- `ResponseEvent` import manquant
- `\Drupal::currentUser()` anti-pattern dans Entity create
- `defaultConfiguration()` absent du Block plugin
- Table `.info.yml` : `config_devel` (module contrib) retiré
- Commandes Drush erronées corrigées
- `api_key: ''` dangereux en config YAML → avertissement
- `token` retiré de la table Form types (#type invalide)

**Ajouts :**
- `FormBase` avec `create()` et DI
- `#states` API
- `_title_callback`
- `hook_preprocess_HOOK`, `hook_page_attachments`, `hook_block_access`, `hook_node_access`, `hook_user_login/logout`, `hook_tokens`
- `#[Hook]` D11 (correction : "RFC en cours" → "disponible")
- Table anti-patterns étendue
- Section "Créer son propre service" (`.services.yml` + classe)

---

## v1.0 — 2026-05-14

**Création initiale**

Couverture :
- Routing + Controllers + DI
- Plugin System (Block complet)
- Hooks vs EventSubscribers
- Forms API + AJAX
- Entity API + EntityQuery
- Config API vs State API
- Cache tags + contexts + max-age

---

## Drupal Compatibility

| Skill version | Drupal testé | PHP minimum |
|--------------|-------------|-------------|
| v3.x | D10, D11 | 8.1 (D10) / 8.3 (D11) |
| v2.x | D10, D11 | 8.1 |
| v1.x | D10 | 8.1 |
