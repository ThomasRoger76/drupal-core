# Leçons — drupal-core

Bugs et patterns découverts en usage réel. Mis à jour après chaque correction.
**Format :** date | composant | symptôme | cause | prévention

---

## 2026-05-14

### `blockAccess()` — visibilité et signature incorrectes
- **Composant :** plugin-system.md, `BlockBase`
- **Symptôme :** PHP Fatal `Declaration of blockAccess() must be compatible with BlockBase::blockAccess()`
- **Cause :** `public` au lieu de `protected`, paramètre `$return_as_object` absent de la vraie signature
- **Correct :** `protected function blockAccess(AccountInterface $account): AccessResultInterface`
- **Prévention :** Toujours vérifier la signature dans `BlockBase.php` avant de surcharger une méthode

### `php: ^8.2` dans `.info.yml` — syntaxe Composer non reconnue
- **Composant :** module-structure.md, `.info.yml`
- **Symptôme :** La contrainte de version PHP est silencieusement ignorée — le module tourne sur n'importe quelle version PHP
- **Cause :** Drupal utilise `version_compare()` sur la valeur string, pas le parser Composer
- **Correct :** `php: '8.2'` (version minimale exacte en string)
- **Prévention :** Ne jamais utiliser `^`, `~`, `>=` dans la clé `php:` du `.info.yml`

### `logger.factory` vs `logger.channel.mon_module` — types incompatibles
- **Composant :** services-internal-api.md, `ArticleManager`
- **Symptôme :** Erreur de type : `@logger.factory` injecte `LoggerChannelFactoryInterface`, pas `LoggerInterface`
- **Cause :** Confusion entre la factory et le channel résultant
- **Correct :** `@logger.channel.mon_module` → `LoggerInterface` directement (PSR-3)
- **Prévention :** Toujours injecter `@logger.channel.NOM_MODULE` pour obtenir `LoggerInterface`

### `time()` PHP natif incohérent avec `TimeInterface`
- **Composant :** database-api.md, UPSERT
- **Symptôme :** Incohérence dans le même fichier — INSERT/UPDATE utilisaient `$this->time`, UPSERT utilisait `time()`
- **Cause :** Correction partielle qui n'a pas couvert tous les exemples
- **Correct :** Systématiquement `$this->time->getRequestTime()` dans les classes, `\Drupal::time()` dans les hooks
- **Prévention :** Chercher `time()` dans tout le fichier après chaque correction liée au temps

### Double `#cache` Block — méthodes de classe ET render array
- **Composant :** plugin-system.md, `BlockBase`
- **Symptôme :** Confusion sur quel mécanisme prend effet — les deux sont redondants
- **Cause :** `getCacheTags()`/`getCacheContexts()` (niveau bloc) et `#cache` dans `build()` (niveau render array) sont deux mécanismes alternatifs
- **Correct :** Pour les blocs, privilégier les méthodes de classe. Retirer `#cache` de `build()` si les méthodes sont définies
- **Prévention :** Ne jamais définir les deux simultanément dans un bloc

### `_custom_access` vs tag `access_check` — approches mutuellement exclusives
- **Composant :** routing-controllers.md
- **Symptôme :** Route 403 inattendue ou service non résolu
- **Cause :** `_custom_access: 'Classe::méthode'` et le tag `{ applies_to: _mon_module_access }` sont deux mécanismes différents — le second requiert `_mon_module_access: 'TRUE'` dans le routing, pas `_custom_access`
- **Prévention :** Choisir l'une des deux approches et rester cohérent

### `ConfigFactory` (classe) injecté au lieu de `ConfigFactoryInterface` (interface)
- **Symptôme :** PhpStan warning, couplage fort à l'implémentation
- **Cause :** `use Drupal\Core\Config\ConfigFactory` au lieu de `ConfigFactoryInterface`
- **Correct :** `private readonly ConfigFactoryInterface $configFactory`
- **Prévention :** Règle universelle DI Drupal — toujours une interface, jamais une classe concrète dans les constructeurs

### Identifiant de service en CamelCase — non standard
- **Symptôme :** `\Drupal::service('mon_module.MyService')` — casse non standard, outils confus
- **Cause :** `mon_module.MyService:` dans `.services.yml` au lieu de `mon_module.my_service:`
- **Correct :** Convention : `vendor.snake_case` — tout en lowercase + underscores
- **Prévention :** Identifiants de services = même règle que les machine names Drupal

### `echo` dans un service Drupal — Output non contrôlable
- **Symptôme :** Output de debug dans des contextes non-CLI, tests cassés, logs absents
- **Cause :** `echo "..."` dans une méthode de service pour afficher la progression
- **Correct :** Injecter `LoggerInterface` via `@logger.channel.mon_module` → `$this->logger->notice()`
- **Prévention :** Un service ne fait jamais d'output direct — il retourne des données ou log. L'output va dans la Drush Command uniquement.

### `AliasManagerInterface` deprecated D10+
- **Composant :** services-internal-api.md, table des services
- **Symptôme :** Deprecation notice sur D10/D11
- **Cause :** Interface refactorisée en D9.3
- **Correct :** `path_alias.repository` + `AliasRepositoryInterface`
- **Prévention :** Vérifier les change records Drupal avant tout upgrade de version majeure

### `??` redondant après `defaultConfiguration()`
- **Composant :** plugin-system.md, `blockForm()` et `build()`
- **Symptôme :** Code trompeur — laisse croire que la valeur peut être null alors que `defaultConfiguration()` la garantit
- **Correct :** Retirer les `?? valeur_defaut` quand `defaultConfiguration()` est définie
- **Prévention :** Toujours définir `defaultConfiguration()` en premier, puis utiliser `$this->configuration['key']` directement

---

## Comment ajouter une leçon

Après chaque correction d'un bug dans du code Drupal utilisant ce skill :

1. Identifier si la cause racine est un gap ou une erreur dans `drupal-core`
2. Ajouter une entrée dans ce fichier avec date + symptôme + cause + prévention
3. Corriger le fichier source concerné
4. Si le bug est systémique, ajouter une ligne dans `CHANGELOG.md`
