---
name: symfony-scheduler
description: "Crée une tâche planifiée Symfony 6.3+ : classe Message readonly, ScheduleProvider avec RecurringMessage (cron ou intervalle), MessageHandler, config messenger.yaml."
argument-hint: "<MessageClass> <CronExpression|interval> — ex: SendDailyReportMessage '0 8 * * *'"
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep"]
---

# Symfony Scheduler

Crée une tâche récurrente avec le composant Scheduler de Symfony 6.3+ (basé sur Messenger).

**Arguments:** "$ARGUMENTS"

Parse les arguments :
- `MessageClass` — nom de la classe message en PascalCase (ex. `SendDailyReportMessage`)
- `CronExpression` — expression cron (ex. `0 8 * * *`) ou intervalle (ex. `1 hour`, `30 minutes`)

Dériver :
- `MessageCamel` — camelCase du message sans "Message" (ex. `sendDailyReport`)
- `HandlerClass` — `{MessageClass}Handler`

---

## Étape 1 — Vérifier les prérequis

Vérifier que le composant Scheduler est installé :

```bash
composer show symfony/scheduler 2>/dev/null || echo "NOT_INSTALLED"
```

Si absent : `composer require symfony/scheduler`

Vérifier le Schedule existant :

```bash
find src/Scheduler -name '*.php' 2>/dev/null
```

---

## Étape 2 — Créer la classe Message

Créer `src/Message/{MessageClass}.php` :

```php
<?php
declare(strict_types=1);

namespace App\Message;

final readonly class SendDailyReportMessage
{
    public function __construct(
        public readonly \DateTimeImmutable $triggeredAt = new \DateTimeImmutable(),
    ) {}
}
```

La classe est `readonly` (PHP 8.2+) : toutes ses propriétés sont immuables.

---

## Étape 3 — Créer ou compléter le Schedule Provider

Si `src/Scheduler/MainSchedule.php` n'existe pas, le créer :

```php
<?php
declare(strict_types=1);

namespace App\Scheduler;

use App\Message\CleanTempFilesMessage;
use App\Message\SendDailyReportMessage;
use Symfony\Component\Scheduler\Attribute\AsSchedule;
use Symfony\Component\Scheduler\RecurringMessage;
use Symfony\Component\Scheduler\Schedule;
use Symfony\Component\Scheduler\ScheduleProviderInterface;
use Symfony\Contracts\Cache\CacheInterface;

#[AsSchedule('main')]
final class MainSchedule implements ScheduleProviderInterface
{
    private ?Schedule $schedule = null;

    public function __construct(private readonly CacheInterface $cache) {}

    public function getSchedule(): Schedule
    {
        return $this->schedule ??= (new Schedule())
            ->add(
                // Expression cron : tous les jours à 8h
                RecurringMessage::cron('0 8 * * *', new SendDailyReportMessage()),
            )
            ->add(
                // Intervalle : toutes les heures
                RecurringMessage::every('1 hour', new CleanTempFilesMessage()),
            )
            ->stateful($this->cache); // Persiste la date du dernier run après redémarrage
    }
}
```

Si `MainSchedule.php` existe déjà, ajouter le `RecurringMessage` dans la méthode `getSchedule()` existante.

**Expressions cron courantes :**
```
'0 8 * * *'      → tous les jours à 8h
'0 */6 * * *'    → toutes les 6 heures
'0 9 * * 1'      → chaque lundi à 9h
'*/15 * * * *'   → toutes les 15 minutes
'0 0 1 * *'      → le 1er de chaque mois à minuit
```

**Intervalles lisibles (RecurringMessage::every) :**
```
'30 minutes'
'1 hour'
'2 hours'
'1 day'
```

---

## Étape 4 — Créer le MessageHandler

Créer `src/MessageHandler/{HandlerClass}.php` :

```php
<?php
declare(strict_types=1);

namespace App\MessageHandler;

use App\Message\SendDailyReportMessage;
use Psr\Log\LoggerInterface;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final class SendDailyReportMessageHandler
{
    public function __construct(
        private readonly LoggerInterface $logger,
        // Injecter ici les services nécessaires au traitement
    ) {}

    public function __invoke(SendDailyReportMessage $message): void
    {
        $this->logger->info('Sending daily report', [
            'triggered_at' => $message->triggeredAt->format(\DateTimeInterface::ATOM),
        ]);

        // Implémenter la logique ici
    }
}
```

---

## Étape 5 — Configurer Messenger

Vérifier `config/packages/messenger.yaml` :

```bash
grep -A 20 'scheduler' config/packages/messenger.yaml 2>/dev/null || echo "NO_SCHEDULER_CONFIG"
```

Si la config scheduler est absente, ajouter dans `config/packages/messenger.yaml` :

```yaml
framework:
    messenger:
        transports:
            # Transport dédié au scheduler (requis)
            scheduler_main:
                dsn: 'symfony://default'

        routing:
            'App\Message\SendDailyReportMessage': scheduler_main
            # Ajouter chaque nouveau message ici
```

---

## Étape 6 — Lancer le worker en développement

```bash
# Consommer le scheduler (Ctrl+C pour arrêter)
php bin/console messenger:consume scheduler_main -vv

# Avec limite de temps (production)
php bin/console messenger:consume scheduler_main --time-limit=3600

# Debugger les messages enregistrés
php bin/console debug:messenger
```

**En production**, superviser le worker avec `supervisord` ou `systemd` :

```ini
; /etc/supervisor/conf.d/messenger-scheduler.conf
[program:messenger-scheduler]
command=php /var/www/app/bin/console messenger:consume scheduler_main --time-limit=3600
autostart=true
autorestart=true
startsecs=5
```

---

## Résultat attendu

Rapporter :
- Fichiers créés (Message, Handler, Schedule mis à jour)
- Config Messenger ajoutée ou vérifiée
- Expression cron ou intervalle utilisé
- Commande pour lancer le worker en dev
