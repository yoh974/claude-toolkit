---
name: symfony-notification
description: "Crée une Notification Symfony complète avec le composant Notifier : classe Notification typée, canaux configurables (email, browser, slack, telegram, sms), config notifier.yaml, et service d'envoi injecté."
argument-hint: "<NotificationName> <channels> — ex: OrderConfirmed email,browser  |  PaymentFailed email,slack,sms"
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep"]
---

# Symfony Notification

Crée une notification Symfony avec le composant `symfony/notifier`.

**Arguments:** "$ARGUMENTS"

Parser les arguments :
- `NotificationName` — nom PascalCase de la notification (ex. `OrderConfirmed`)
- `channels` — liste séparée par des virgules parmi : `email`, `browser`, `slack`, `telegram`, `sms` (ex. `email,browser` ou `email,slack,sms`)

Dériver :
- `notification_name` — snake_case du nom (ex. `order_confirmed`)
- `NotificationSubject` — titre lisible (ex. `Order Confirmed` → à adapter selon le domaine)
- Pour chaque canal : déterminer le bridge package et l'interface correspondante

---

## Étape 1 — Découverte du projet

```bash
# Vérifier si symfony/notifier est installé
composer show symfony/notifier 2>/dev/null || echo "NOT_INSTALLED"

# Lister les notifications existantes pour respecter les conventions
find src/Notification -name '*.php' 2>/dev/null
find src -name '*Notification.php' 2>/dev/null | head -5

# Vérifier la config notifier existante
cat config/packages/notifier.yaml 2>/dev/null || echo "NO_CONFIG"
```

Si `symfony/notifier` absent : `composer require symfony/notifier`

Pour chaque canal demandé, vérifier le bridge correspondant :

| Canal | Package | Commande |
|-------|---------|----------|
| `email` | `symfony/mailer` | `composer require symfony/mailer` |
| `browser` | intégré dans notifier | (rien) |
| `slack` | `symfony/slack-notifier` | `composer require symfony/slack-notifier` |
| `telegram` | `symfony/telegram-notifier` | `composer require symfony/telegram-notifier` |
| `sms` | `symfony/twilio-notifier` | `composer require symfony/twilio-notifier` |

---

## Étape 2 — Créer la classe Notification

Créer `src/Notification/{NotificationName}Notification.php`.

Adapter le contenu selon les canaux demandés.

### Template complet (tous canaux — supprimer les blocs non demandés)

```php
<?php

declare(strict_types=1);

namespace App\Notification;

use Symfony\Bridge\Twig\Mime\TemplatedEmail;
use Symfony\Component\Mailer\MailerInterface;
use Symfony\Component\Notifier\Bridge\Slack\Block\SlackDividerBlock;
use Symfony\Component\Notifier\Bridge\Slack\Block\SlackSectionBlock;
use Symfony\Component\Notifier\Bridge\Slack\SlackOptions;
use Symfony\Component\Notifier\Bridge\Telegram\TelegramOptions;
use Symfony\Component\Notifier\Bridge\Twilio\TwilioOptions;
use Symfony\Component\Notifier\Message\ChatMessage;
use Symfony\Component\Notifier\Message\EmailMessage;
use Symfony\Component\Notifier\Message\SmsMessage;
use Symfony\Component\Notifier\Notification\Notification;
use Symfony\Component\Notifier\Recipient\EmailRecipientInterface;
use Symfony\Component\Notifier\Recipient\RecipientInterface;
use Symfony\Component\Notifier\Recipient\SmsRecipientInterface;

final class OrderConfirmedNotification extends Notification
{
    public function __construct(
        private readonly string $orderReference,
        private readonly float  $total,
    ) {
        parent::__construct('Order Confirmed');
    }

    /**
     * Déclare les canaux actifs selon le destinataire disponible.
     * Symfony Notifier n'envoie que sur les canaux supportés par le recipient.
     *
     * @return list<string>
     */
    public function getChannels(RecipientInterface $recipient): array
    {
        $channels = ['browser']; // toujours actif (flash message)

        if ($recipient instanceof EmailRecipientInterface) {
            $channels[] = 'email';
        }

        if ($recipient instanceof SmsRecipientInterface) {
            $channels[] = 'sms/twilio';
        }

        return $channels;
    }

    // -------------------------------------------------------------------------
    // Canal : browser (flash message — pas de méthode dédiée, géré par Notifier)
    // Le sujet de la Notification est utilisé comme message flash.
    // -------------------------------------------------------------------------

    // -------------------------------------------------------------------------
    // Canal : email
    // -------------------------------------------------------------------------
    public function asEmailMessage(EmailRecipientInterface $recipient, string $transport = null): ?EmailMessage
    {
        $email = (new TemplatedEmail())
            ->to($recipient->getEmail())
            ->subject('Commande confirmée — ' . $this->orderReference)
            ->htmlTemplate('emails/order_confirmed.html.twig')
            ->context([
                'order_reference' => $this->orderReference,
                'total'           => $this->total,
            ]);

        return new EmailMessage($email);
    }

    // -------------------------------------------------------------------------
    // Canal : slack
    // -------------------------------------------------------------------------
    public function asChatMessage(RecipientInterface $recipient, string $transport = null): ?ChatMessage
    {
        if ('slack' !== $transport) {
            return null;
        }

        $options = (new SlackOptions())
            ->block((new SlackSectionBlock())->text('*Nouvelle commande confirmée*'))
            ->block((new SlackSectionBlock())->text("Référence : `{$this->orderReference}`\nTotal : *{$this->total} €*"))
            ->block(new SlackDividerBlock());

        return (new ChatMessage('Order confirmed: ' . $this->orderReference))
            ->options($options);
    }

    // -------------------------------------------------------------------------
    // Canal : telegram
    // -------------------------------------------------------------------------
    // public function asChatMessage(RecipientInterface $recipient, string $transport = null): ?ChatMessage
    // {
    //     if ('telegram' !== $transport) {
    //         return null;
    //     }
    //
    //     $options = (new TelegramOptions())
    //         ->chatId('%env(TELEGRAM_CHAT_ID)%')
    //         ->parseMode('MarkdownV2');
    //
    //     return (new ChatMessage("*Commande confirmée*\nRéférence : `{$this->orderReference}`"))
    //         ->options($options);
    // }

    // -------------------------------------------------------------------------
    // Canal : sms
    // -------------------------------------------------------------------------
    public function asSmsMessage(SmsRecipientInterface $recipient, string $transport = null): ?SmsMessage
    {
        return new SmsMessage(
            $recipient->getPhone(),
            "Commande {$this->orderReference} confirmée. Total : {$this->total} €."
        );
    }
}
```

**Adapter obligatoirement :**
- Le constructeur avec les données métier réelles de la notification
- Le sujet (`'Order Confirmed'`) → libellé métier correct
- `asEmailMessage()` → template Twig + contexte réel
- `asChatMessage()` → texte et blocs Slack/Telegram selon les données
- `asSmsMessage()` → message court (< 160 chars si possible)
- `getChannels()` → supprimer les canaux non demandés

---

## Étape 3 — Créer le template email (si canal `email`)

Créer `templates/emails/{notification_name}.html.twig` :

```twig
{% extends 'emails/base.html.twig' %}

{% block content %}
    <h1>Commande confirmée</h1>
    <p>Référence : <strong>{{ order_reference }}</strong></p>
    <p>Total : <strong>{{ total }} €</strong></p>
{% endblock %}
```

Si `templates/emails/base.html.twig` n'existe pas, créer un template autonome minimal.

---

## Étape 4 — Configurer `notifier.yaml`

Compléter ou créer `config/packages/notifier.yaml` selon les canaux demandés :

```yaml
framework:
    notifier:
        # Canal browser : active les flash messages
        channel_policy:
            # Politique par importance : urgent, high, medium, low
            urgent: ['email', 'sms/twilio']
            high:   ['email', 'slack/slack_notifier']
            medium: ['email', 'browser']
            low:    ['browser']

        # Chatter transports (Slack, Telegram)
        chatter_transports:
            slack_notifier: '%env(SLACK_DSN)%'
            # telegram_notifier: '%env(TELEGRAM_DSN)%'

        # SMS transports
        texter_transports:
            twilio: '%env(TWILIO_DSN)%'
```

Ajouter les variables d'environnement manquantes dans `.env` :

```dotenv
# Slack — format: slack://TOKEN@default?channel=CHANNEL_NAME
SLACK_DSN=slack://xoxb-your-token@default?channel=notifications

# Twilio — format: twilio://SID:TOKEN@default?from=+33600000000
TWILIO_DSN=twilio://ACCOUNT_SID:AUTH_TOKEN@default?from=%2B33600000000

# Telegram — format: telegram://TOKEN@default?channel=CHAT_ID
# TELEGRAM_DSN=telegram://BOT_TOKEN@default?channel=@mychannel
```

---

## Étape 5 — Injecter et utiliser dans un Service

```php
<?php

declare(strict_types=1);

namespace App\Service;

use App\Notification\OrderConfirmedNotification;
use Symfony\Component\Notifier\NotifierInterface;
use Symfony\Component\Notifier\Recipient\Recipient;

final class OrderService
{
    public function __construct(
        private readonly NotifierInterface $notifier,
    ) {}

    public function confirmOrder(Order $order): void
    {
        // … logique métier …

        $notification = new OrderConfirmedNotification(
            orderReference: $order->getReference(),
            total:          $order->getTotal(),
        );

        $recipient = new Recipient(
            email: $order->getUser()->getEmail(),
            phone: $order->getUser()->getPhone() ?? '',
        );

        $this->notifier->send($notification, $recipient);
    }
}
```

**`Recipient`** implémente à la fois `EmailRecipientInterface` et `SmsRecipientInterface`.
Si le phone est vide (`''`), le canal SMS est ignoré automatiquement.

---

## Étape 6 — Flash message dans un Controller (canal `browser`)

Le canal `browser` ajoute automatiquement un message flash. Pour l'afficher dans Twig :

```twig
{# templates/base.html.twig #}
{% for type, messages in app.flashes(['success', 'warning', 'error']) %}
    {% for message in messages %}
        <div class="alert alert-{{ type }}">{{ message }}</div>
    {% endfor %}
{% endfor %}
```

---

## Étape 7 — Vérification

```bash
# Vérifier les transports configurés
php bin/console debug:config framework notifier

# Lister les services Notifier disponibles
php bin/console debug:autowiring notifier

# Vider le cache
php bin/console cache:clear
```

---

## Résultat attendu

Rapporter :
- Fichier `Notification` créé avec les canaux actifs
- Template email créé (si canal `email`)
- Variables d'environnement à renseigner dans `.env`
- Exemple d'utilisation dans le service concerné
- Canaux ignorés car bridge absent (indiquer la commande `composer require` correspondante)
