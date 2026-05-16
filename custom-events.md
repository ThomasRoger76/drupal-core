---
name: drupal-core — custom events
description: Créer et dispatcher ses propres événements Drupal avec le système EventDispatcher Symfony - classe Event, dispatch, subscribers, et découplage des modules.
---

# Custom Events — Créer ses Propres Événements Drupal

## Pourquoi Créer des Événements Custom

```
Hook Drupal (hook_entity_presave) :
  → Couplage fort : tout module qui veut réagir doit connaître le hook
  → Impossible de passer des données structurées complexes
  → Pas de résultat retourné possible

Custom Event avec EventDispatcher :
  ✅ Découplage total entre le dispatcher et les subscribers
  ✅ Données typées (classe Event avec getters/setters)
  ✅ Stoppable (arrêter la propagation)
  ✅ Testable (mock de l'EventDispatcher)
  ✅ Pattern Drupal natif depuis D8
```

---

## Étape 1 — Créer la Classe Event

```php
<?php
// src/Event/CommandeCreeeEvent.php
namespace Drupal\mon_module\Event;

use Drupal\mon_module\Entity\Commande;
use Symfony\Contracts\EventDispatcher\Event;

/**
 * Événement déclenché après la création d'une commande.
 *
 * Dispatcher : CommandeService::creerCommande()
 * Subscribers potentiels : module CRM, module Email, module Analytics
 */
class CommandeCreeeEvent extends Event {

  /**
   * Nom de l'événement — convention : nom_module.action_effectuée
   */
  const EVENT_NAME = 'mon_module.commande_creee';

  public function __construct(
    private readonly Commande $commande,
    private bool $notificationEnvoyee = FALSE,
  ) {}

  public function getCommande(): Commande {
    return $this->commande;
  }

  public function isNotificationEnvoyee(): bool {
    return $this->notificationEnvoyee;
  }

  public function setNotificationEnvoyee(bool $value): void {
    $this->notificationEnvoyee = $value;
  }
}
```

---

## Étape 2 — Dispatcher l'Événement

```php
<?php
// src/Service/CommandeService.php
namespace Drupal\mon_module\Service;

use Drupal\mon_module\Event\CommandeCreeeEvent;
use Symfony\Component\EventDispatcher\EventDispatcherInterface;

class CommandeService {

  public function __construct(
    private readonly EventDispatcherInterface $eventDispatcher,
  ) {}

  public function creerCommande(array $data): Commande {
    // Créer et sauvegarder la commande
    $commande = Commande::create($data);
    $commande->save();

    // Dispatcher l'événement — tous les subscribers sont notifiés
    $event = new CommandeCreeeEvent($commande);
    $this->eventDispatcher->dispatch($event, CommandeCreeeEvent::EVENT_NAME);

    // Lire le résultat enrichi par les subscribers
    if ($event->isNotificationEnvoyee()) {
      \Drupal::logger('mon_module')->info('Email envoyé pour commande @id', ['@id' => $commande->id()]);
    }

    return $commande;
  }
}
```

```yaml
# mon_module.services.yml
services:
  mon_module.commande_service:
    class: Drupal\mon_module\Service\CommandeService
    arguments:
      - '@event_dispatcher'
```

---

## Étape 3 — Créer un Subscriber

```php
<?php
// src/EventSubscriber/CommandeEmailSubscriber.php
namespace Drupal\mon_module\EventSubscriber;

use Drupal\mon_module\Event\CommandeCreeeEvent;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

class CommandeEmailSubscriber implements EventSubscriberInterface {

  public function __construct(
    private readonly \Drupal\symfony_mailer\EmailFactoryInterface $emailFactory,
  ) {}

  public static function getSubscribedEvents(): array {
    return [
      CommandeCreeeEvent::EVENT_NAME => [
        ['envoyerConfirmation', 10],  // Priorité 10 (plus haute = première)
      ],
    ];
  }

  public function envoyerConfirmation(CommandeCreeeEvent $event): void {
    $commande = $event->getCommande();

    $this->emailFactory
      ->newTypedEmail('mon_module', 'commande_confirmee', $commande)
      ->send();

    // Enrichir l'événement avec le résultat
    $event->setNotificationEnvoyee(TRUE);
  }
}
```

```yaml
# mon_module.services.yml — déclarer le subscriber
services:
  mon_module.commande_email_subscriber:
    class: Drupal\mon_module\EventSubscriber\CommandeEmailSubscriber
    arguments:
      - '@email_factory'
    tags:
      - { name: event_subscriber }
```

---

## Événements Stoppables

```php
// Événement qui peut être interrompu (aucun autre subscriber exécuté après)
use Symfony\Contracts\EventDispatcher\Event;

class CommandeValidationEvent extends Event {
  const EVENT_NAME = 'mon_module.commande_validation';

  private array $erreurs = [];

  public function __construct(
    private readonly array $donnees,
  ) {}

  public function getDonnees(): array {
    return $this->donnees;
  }

  public function ajouterErreur(string $erreur): void {
    $this->erreurs[] = $erreur;
    // Stopper la propagation si validation échoue
    $this->stopPropagation();
  }

  public function getErreurs(): array {
    return $this->erreurs;
  }

  public function isValide(): bool {
    return empty($this->erreurs);
  }
}

// Dans le service — vérifier si stoppé
$event = new CommandeValidationEvent($donnees);
$this->eventDispatcher->dispatch($event, CommandeValidationEvent::EVENT_NAME);
if (!$event->isValide()) {
  throw new \InvalidArgumentException(implode(', ', $event->getErreurs()));
}
```

---

## Tester un Custom Event (PHPUnit)

```php
// tests/src/Unit/EventSubscriber/CommandeEmailSubscriberTest.php
use Drupal\Tests\UnitTestCase;

class CommandeEmailSubscriberTest extends UnitTestCase {

  public function testEnvoyerConfirmationSetsFlag(): void {
    $emailFactory = $this->createMock(\Drupal\symfony_mailer\EmailFactoryInterface::class);
    $commande = $this->createMock(\Drupal\mon_module\Entity\Commande::class);

    $emailFactory->expects($this->once())->method('newTypedEmail');

    $subscriber = new CommandeEmailSubscriber($emailFactory);
    $event = new CommandeCreeeEvent($commande);

    $subscriber->envoyerConfirmation($event);

    $this->assertTrue($event->isNotificationEnvoyee());
  }
}
```

---

## Events Drupal Core Disponibles

```php
// Events courants à connaître — les "hooks déguisés"
use Drupal\Core\Entity\EntityEvents;
// EntityEvents::BUNDLE_CREATE, BUNDLE_DELETE, FIELD_STORAGE_CREATE...

use Drupal\Core\Config\ConfigEvents;
// ConfigEvents::SAVE, ConfigEvents::DELETE, ConfigEvents::IMPORT

use Symfony\Component\HttpKernel\KernelEvents;
// KernelEvents::REQUEST, RESPONSE, TERMINATE, EXCEPTION...

use Drupal\Core\Routing\RoutingEvents;
// RoutingEvents::ALTER, DYNAMIC, FINISHED

use Drupal\Core\Language\LanguageEvents;
// LanguageEvents::LANGUAGE_TYPES_CONFIGURE_FINISHED
```
