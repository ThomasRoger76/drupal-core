# Symfony Mailer — Emails Drupal 11 (remplace SwiftMailer)

## Contexte

SwiftMailer a été retiré de Drupal 10/11. Le module `symfony_mailer` (contrib) utilise le composant `symfony/mailer` pour l'envoi d'emails en D10/D11. Il remplace `hook_mail` + `drupal_mail()` par un système basé sur des plugins PHP.

```bash
composer require drupal/symfony_mailer
docker compose exec php drush en symfony_mailer -y
```

---

## Architecture

```
EmailFactory::newTypedEmail()
  → EmailBuilder plugin (construit le contenu)
  → EmailAdjuster plugins (modifient l'email : sujet, from, headers)
  → Transport (SMTP, Sendmail, DSN)
```

---

## Envoyer un email simple

### Via `email_factory` (méthode recommandée D11)

```php
use Drupal\symfony_mailer\EmailFactoryInterface;

// Injection du service dans une classe
public function __construct(
  private readonly EmailFactoryInterface $emailFactory,
) {}

// Dans une méthode
$email = $this->emailFactory
  ->newTypedEmail('mon_module', 'confirmation')
  ->setTo('destinataire@exemple.com')
  ->setParam('user', $user)
  ->setParam('confirmation_url', $confirmation_url);

$email->send();
```

### Via le service statique (à éviter, mais disponible)

```php
\Drupal::service('email_factory')
  ->newTypedEmail('mon_module', 'notification')
  ->setTo($account->getEmail())
  ->setParam('node', $node)
  ->send();
```

---

## Définir un EmailBuilder plugin

```php
// src/Plugin/EmailBuilder/ConfirmationEmailBuilder.php
namespace Drupal\mon_module\Plugin\EmailBuilder;

use Drupal\symfony_mailer\Annotation\EmailBuilder;
use Drupal\symfony_mailer\EmailInterface;
use Drupal\symfony_mailer\Processor\EmailBuilderBase;

/**
 * Email builder pour les confirmations d'inscription.
 *
 * @EmailBuilder(
 *   id = "mon_module",
 *   sub_types = {
 *     "confirmation" = @Translation("Confirmation inscription"),
 *     "reset_password" = @Translation("Réinitialisation mot de passe"),
 *   },
 *   label = @Translation("Mon Module"),
 *   import = TRUE,
 *   override = {},
 * )
 */
class ConfirmationEmailBuilder extends EmailBuilderBase {

  /**
   * {@inheritdoc}
   *
   * Initialise les paramètres nécessaires avant le build.
   */
  public function createParams(EmailInterface $email, ?object $param = NULL): void {
    if ($param instanceof UserInterface) {
      $email->setParam('user', $param);
    }
  }

  /**
   * {@inheritdoc}
   *
   * Construit le contenu de l'email.
   */
  public function build(EmailInterface $email): void {
    $user = $email->getParam('user');
    $confirmation_url = $email->getParam('confirmation_url');

    $email
      ->setSubject((string) $this->t(
        'Confirmez votre inscription sur @site',
        ['@site' => \Drupal::config('system.site')->get('name')]
      ))
      ->setBody([
        '#theme' => 'email_confirmation',
        '#user' => $user,
        '#confirmation_url' => $confirmation_url,
      ]);
  }

}
```

### Builder D11 avec PHP Attributes

```php
// Drupal 11 — PHP Attributes (équivalent sans annotation)
use Drupal\symfony_mailer\Processor\EmailBuilderBase;

#[EmailBuilder(
  id: 'mon_module',
  sub_types: ['confirmation' => 'Confirmation inscription'],
  label: 'Mon Module',
)]
class ConfirmationEmailBuilder extends EmailBuilderBase {

  public function build(EmailInterface $email): void {
    $email
      ->setSubject((string) $this->t('Confirmez votre inscription'))
      ->setBody([
        '#theme' => 'email_confirmation',
        '#user' => $email->getParam('user'),
        '#link' => $email->getParam('confirmation_url'),
      ]);
  }

}
```

---

## Template Twig pour un email

### Déclaration du thème

```php
// mon_module.module
function mon_module_theme(): array {
  return [
    'email_confirmation' => [
      'variables' => [
        'user' => NULL,
        'confirmation_url' => NULL,
      ],
    ],
  ];
}
```

### Template Twig

```twig
{# templates/email-confirmation.html.twig #}
{% extends '@symfony_mailer/base.html.twig' %}

{% block subject %}
  {{ 'Confirmez votre inscription'|t }}
{% endblock %}

{% block body_html %}
<html>
<body style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">

  <h1 style="color: #333;">{{ 'Bienvenue !'|t }}</h1>

  <p>
    {{ 'Bonjour @name,'|t({'@name': user.displayName}) }}
  </p>

  <p>
    {{ 'Merci de vous être inscrit. Veuillez confirmer votre compte en cliquant sur le lien ci-dessous.'|t }}
  </p>

  <p style="text-align: center; margin: 30px 0;">
    <a href="{{ confirmation_url }}"
       style="background: #0073ba; color: white; padding: 12px 24px;
              text-decoration: none; border-radius: 4px; display: inline-block;">
      {{ 'Confirmer mon compte'|t }}
    </a>
  </p>

  <p style="color: #666; font-size: 12px;">
    {{ 'Ce lien expire dans 24 heures.'|t }}
  </p>

</body>
</html>
{% endblock %}

{% block body_plain %}
{{ 'Bonjour @name,'|t({'@name': user.displayName}) }}

{{ 'Confirmez votre compte : @url'|t({'@url': confirmation_url}) }}
{% endblock %}
```

---

## Configuration SMTP

### Via settings.php (recommandé pour la production)

```php
// settings.php ou settings.local.php
$config['symfony_mailer.mailer_transport.smtp']['configuration'] = [
  'host' => 'smtp.monserveur.com',
  'port' => 587,
  'user' => 'noreply@monsite.com',
  'pass' => 'mot_de_passe_smtp',
  'encryption' => 'tls',  // ou 'ssl', 'none'
  'timeout' => 30,
];
```

### Via DSN (Symfony Mailer natif)

```php
// Utiliser un DSN complet
$config['symfony_mailer.mailer_transport.smtp']['configuration']['dsn'] =
  'smtp://user:pass@smtp.exemple.com:587?encryption=tls';
```

### Via interface admin

`/admin/config/system/mailer/transport` → Configurer le transport SMTP.

---

## Tests email — MailHog en développement

### docker-compose.yml

```yaml
services:
  mailhog:
    image: mailhog/mailhog:latest
    ports:
      - "1025:1025"   # SMTP
      - "8025:8025"   # Interface web
    networks:
      - drupal
```

### Configuration Drupal pour MailHog

```php
// settings.local.php
$config['symfony_mailer.mailer_transport.smtp']['configuration'] = [
  'host' => 'mailhog',  // nom du service Docker
  'port' => 1025,
  'user' => '',
  'pass' => '',
  'encryption' => 'none',
];
```

Interface web MailHog : `http://localhost:8025`

### Capturer les emails dans les tests PHPUnit

```php
// Dans un KernelTestBase ou FunctionalTestBase
use Drupal\Core\Test\AssertMailTrait;

class MonEmailTest extends KernelTestBase {
  use AssertMailTrait;

  protected function setUp(): void {
    parent::setUp();
    // Activer le collecteur d'emails de test
    $this->config('system.mail')
      ->set('interface.default', 'test_mail_collector')
      ->save();
  }

  public function testConfirmationEmail(): void {
    // Déclencher l'envoi
    $this->emailFactory
      ->newTypedEmail('mon_module', 'confirmation')
      ->setTo('test@exemple.com')
      ->setParam('user', $this->user)
      ->send();

    // Assertions
    $emails = $this->getMails();
    $this->assertCount(1, $emails);
    $this->assertMailString('to', 'test@exemple.com', $emails);
    $this->assertMailString('subject', 'Confirmez votre inscription', $emails);
  }
}
```

---

## Migration depuis hook_mail (D7/D8/D9 → D11)

### Avant (D7/D8/D9)

```php
// mon_module.module
function mon_module_mail($key, &$message, $params) {
  switch ($key) {
    case 'confirmation':
      $message['subject'] = t('Confirmation');
      $message['body'][] = t('Bonjour @name', ['@name' => $params['user']->getDisplayName()]);
      $message['body'][] = $params['link'];
      break;
  }
}

// Envoi
\Drupal::service('plugin.manager.mail')->mail(
  'mon_module',
  'confirmation',
  $user->getEmail(),
  $user->getPreferredLangcode(),
  ['user' => $user, 'link' => $url]
);
```

### Après (D11 — Symfony Mailer)

```php
// Supprimer hook_mail() et utiliser EmailBuilder
$this->emailFactory
  ->newTypedEmail('mon_module', 'confirmation')
  ->setTo($user->getEmail())
  ->setParam('user', $user)
  ->setParam('confirmation_url', $url)
  ->send();
```

**Points de vigilance lors de la migration :**
- Les hooks `hook_mail_alter()` deviennent des **EmailAdjuster plugins**
- Le sujet et le corps basculent dans le `EmailBuilder::build()`
- Les templates texte brut remplacent les `$message['body']` tableaux

---

## EmailAdjuster — Modifier tous les emails du module

```php
// src/Plugin/EmailAdjuster/AddReplyTo.php
use Drupal\symfony_mailer\Processor\EmailAdjusterBase;

/**
 * @EmailAdjuster(
 *   id = "mon_module_reply_to",
 *   label = @Translation("Ajouter Reply-To support"),
 *   description = @Translation("Ajoute l'adresse support en Reply-To"),
 * )
 */
class AddReplyTo extends EmailAdjusterBase {

  public function process(EmailInterface $email): void {
    $email->addReplyTo('support@monsite.com', 'Support');
  }

}
```

---

## Anti-patterns Symfony Mailer

| ❌ À éviter | ✅ Bonne pratique | Raison |
|------------|------------------|--------|
| `drupal_mail()` en D10/D11 | `email_factory->newTypedEmail()` | `drupal_mail()` passe par l'ancien système |
| HTML non inliné dans les emails | Utiliser `@symfony_mailer/base.html.twig` | Les clients mail ignorent les `<style>` externes |
| Envoyer depuis `hook_entity_presave` synchrone | Utiliser la Queue API | Bloque la requête, risque de timeout SMTP |
| Stocker des credentials SMTP dans le config Drupal exporté | `settings.php` pour les secrets | Les credentials finissent dans le repo git |
| Oublier `body_plain` dans le template | Toujours fournir le fallback texte brut | Les clients sans HTML (mutt, etc.) reçoivent un email vide |

---

## See Also

- `drupal-core` — Plugin system, Services, Queue API
- `drupal-docker` — MailHog dans Docker Compose
- `drupal-testing` — AssertMailTrait, KernelTests
