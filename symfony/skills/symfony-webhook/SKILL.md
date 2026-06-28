---
name: symfony-webhook
description: "Implémente un récepteur webhook Symfony 6.3+ : RequestParser avec vérification de signature, RemoteEvent typé, Consumer avec #[AsRemoteEventConsumer], config webhook.yaml."
argument-hint: "<provider-name> — ex: stripe github slack"
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep"]
---

# Symfony Webhook

Implémente un récepteur webhook avec le composant Webhook Symfony 6.3+.

**Arguments:** "$ARGUMENTS"

Parse les arguments :
- `provider` — nom du provider en minuscules (ex. `stripe`, `github`, `slack`)

Dériver :
- `ProviderPascal` — PascalCase du provider (ex. `Stripe`)
- Route auto-générée : `/webhook/{provider}` (ex. `/webhook/stripe`)

---

## Étape 1 — Vérifier les prérequis

```bash
composer show symfony/webhook 2>/dev/null || echo "NOT_INSTALLED"
composer show symfony/remote-event 2>/dev/null || echo "REMOTE_EVENT_NOT_INSTALLED"
```

Si absents : `composer require symfony/webhook`

Vérifier les webhooks existants :

```bash
find src/Webhook -name '*.php' 2>/dev/null
find src/RemoteEvent -name '*.php' 2>/dev/null
```

---

## Étape 2 — Créer le RemoteEvent

Créer `src/RemoteEvent/{ProviderPascal}WebhookEvent.php` :

```php
<?php
declare(strict_types=1);

namespace App\RemoteEvent;

use Symfony\Component\RemoteEvent\RemoteEvent;

/**
 * Typed wrapper around the raw Stripe webhook payload.
 * RemoteEvent exposes: getName() (event type), getId() (idempotency key), getPayload().
 */
final class StripeWebhookEvent extends RemoteEvent
{
    public function getPaymentIntentId(): string
    {
        return $this->getPayload()['data']['object']['id'] ?? '';
    }

    public function getCustomerId(): string
    {
        return $this->getPayload()['data']['object']['customer'] ?? '';
    }
}
```

Adapter les accesseurs aux champs réels du payload du provider.

---

## Étape 3 — Créer le RequestParser

Créer `src/Webhook/{ProviderPascal}RequestParser.php` :

```php
<?php
declare(strict_types=1);

namespace App\Webhook;

use App\RemoteEvent\StripeWebhookEvent;
use Symfony\Component\HttpFoundation\ChainRequestMatcher;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\RequestMatcher\IsJsonRequestMatcher;
use Symfony\Component\HttpFoundation\RequestMatcher\MethodRequestMatcher;
use Symfony\Component\HttpFoundation\RequestMatcherInterface;
use Symfony\Component\RemoteEvent\RemoteEvent;
use Symfony\Component\Webhook\Client\AbstractRequestParser;
use Symfony\Component\Webhook\Exception\RejectWebhookException;

final class StripeRequestParser extends AbstractRequestParser
{
    protected function getRequestMatcher(): RequestMatcherInterface
    {
        return new ChainRequestMatcher([
            new MethodRequestMatcher('POST'),
            new IsJsonRequestMatcher(),
        ]);
    }

    protected function doParse(Request $request, #[\SensitiveParameter] string $secret): RemoteEvent
    {
        // --- Vérification de la signature Stripe ---
        $signature = $request->headers->get('Stripe-Signature') ?? '';
        if (!$this->verifyStripeSignature($request->getContent(), $signature, $secret)) {
            throw new RejectWebhookException(Response::HTTP_FORBIDDEN, 'Invalid Stripe signature.');
        }

        // --- Parsing du payload ---
        $payload = $request->toArray();
        $eventType = $payload['type'] ?? throw new RejectWebhookException(400, 'Missing event type.');
        $eventId   = $payload['id']   ?? throw new RejectWebhookException(400, 'Missing event id.');

        return new StripeWebhookEvent($eventType, $eventId, $payload);
    }

    private function verifyStripeSignature(string $payload, string $signatureHeader, string $secret): bool
    {
        // Format Stripe: "t=timestamp,v1=signature"
        $parts = [];
        foreach (explode(',', $signatureHeader) as $part) {
            [$key, $value] = explode('=', $part, 2) + ['', ''];
            $parts[$key] = $value;
        }

        $timestamp = $parts['t'] ?? '';
        $signature = $parts['v1'] ?? '';

        if (empty($timestamp) || empty($signature)) {
            return false;
        }

        $expected = hash_hmac('sha256', $timestamp . '.' . $payload, $secret);

        return hash_equals($expected, $signature);
    }
}
```

**Adapter la vérification de signature selon le provider :**

```php
// GitHub : header 'X-Hub-Signature-256'
$signature = $request->headers->get('X-Hub-Signature-256') ?? '';
$expected  = 'sha256=' . hash_hmac('sha256', $request->getContent(), $secret);
return hash_equals($expected, $signature);

// Slack : header 'X-Slack-Signature' + timestamp anti-replay
$timestamp = $request->headers->get('X-Slack-Request-Timestamp') ?? '';
$baseString = "v0:{$timestamp}:" . $request->getContent();
$expected   = 'v0=' . hash_hmac('sha256', $baseString, $secret);
return hash_equals($expected, $request->headers->get('X-Slack-Signature', ''));
```

---

## Étape 4 — Enregistrer dans la config

Créer ou compléter `config/packages/webhook.yaml` :

```yaml
framework:
    webhook:
        routing:
            stripe:  # → génère automatiquement la route POST /webhook/stripe
                service: App\Webhook\StripeRequestParser
                secret: '%env(STRIPE_WEBHOOK_SECRET)%'
```

Ajouter la variable dans `.env` :

```dotenv
STRIPE_WEBHOOK_SECRET=whsec_your_secret_here
```

---

## Étape 5 — Créer le Consumer

Créer `src/RemoteEvent/{ProviderPascal}WebhookConsumer.php` :

```php
<?php
declare(strict_types=1);

namespace App\RemoteEvent;

use Psr\Log\LoggerInterface;
use Symfony\Component\RemoteEvent\Attribute\AsRemoteEventConsumer;
use Symfony\Component\RemoteEvent\Consumer\ConsumerInterface;
use Symfony\Component\RemoteEvent\RemoteEvent;

#[AsRemoteEventConsumer('stripe')]  // Doit correspondre au nom dans webhook.yaml
final class StripeWebhookConsumer implements ConsumerInterface
{
    public function __construct(
        private readonly LoggerInterface $logger,
        // Injecter ici les services nécessaires (OrderService, etc.)
    ) {}

    public function consume(RemoteEvent $event): void
    {
        if (!$event instanceof StripeWebhookEvent) {
            return;
        }

        $this->logger->info('Stripe webhook received', [
            'type'       => $event->getName(),
            'event_id'   => $event->getId(),
        ]);

        match ($event->getName()) {
            'payment_intent.succeeded'       => $this->handlePaymentSucceeded($event),
            'payment_intent.payment_failed'  => $this->handlePaymentFailed($event),
            'customer.subscription.deleted'  => $this->handleSubscriptionCancelled($event),
            default                          => null, // événements non gérés ignorés
        };
    }

    private function handlePaymentSucceeded(StripeWebhookEvent $event): void
    {
        // Mettre à jour la commande, envoyer email de confirmation, etc.
    }

    private function handlePaymentFailed(StripeWebhookEvent $event): void
    {
        // Notifier le client, marquer la commande en erreur, etc.
    }

    private function handleSubscriptionCancelled(StripeWebhookEvent $event): void
    {
        // Désactiver l'accès, notifier, etc.
    }
}
```

---

## Étape 6 — Vérifier et tester

```bash
# Vérifier que la route a été créée
php bin/console debug:router | grep webhook

# Vider le cache
php bin/console cache:clear

# Tester avec curl (exemple Stripe)
curl -X POST http://localhost:8000/webhook/stripe \
  -H "Content-Type: application/json" \
  -H "Stripe-Signature: t=$(date +%s),v1=test_signature" \
  -d '{"type":"payment_intent.succeeded","id":"evt_test_123","data":{"object":{"id":"pi_123"}}}'
```

---

## Résultat attendu

Rapporter :
- Fichiers créés (RemoteEvent, RequestParser, Consumer)
- Route webhook générée
- Variable d'environnement à configurer
- Exemple curl pour tester
