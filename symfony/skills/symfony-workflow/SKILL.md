---
name: symfony-workflow
description: "Configure un Symfony Workflow ou StateMachine : YAML config complet, intégration entity, injection WorkflowInterface, guard listeners avec #[AsEventListener]."
argument-hint: "<workflow-name> <EntityClass> — ex: order-process Order"
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep"]
---

# Symfony Workflow

Configure un Workflow ou StateMachine Symfony avec config YAML, intégration Doctrine et listeners.

**Arguments:** "$ARGUMENTS"

Parse les arguments :
- `workflow-name` — nom en kebab-case (ex. `order-process`)
- `EntityClass` — classe de l'entité Doctrine (ex. `Order`)

Dériver :
- `workflowNameCamel` — camelCase sans tirets (ex. `orderProcess`)
- `workflowNameSnake` — snake_case (ex. `order_process`)
- `entityCamel` — camelCase de l'entity (ex. `order`)

---

## Étape 1 — Choisir le type

**StateMachine** : l'objet se trouve à **exactement un état** à la fois (plus fréquent pour les statuts de commandes, publications, dossiers).

**Workflow** : l'objet peut être dans **plusieurs états simultanés** (ex. processus de validation avec étapes parallèles).

Si incertain, demander à l'utilisateur ou choisir `state_machine` (le cas le plus courant).

---

## Étape 2 — Vérifier les prérequis

```bash
composer show symfony/workflow 2>/dev/null || echo "NOT_INSTALLED"
```

Si absent : `composer require symfony/workflow`

Chercher l'entity :

```bash
grep -r "class {EntityClass}" src/ --include="*.php" -l | head -3
```

---

## Étape 3 — Ajouter le champ de statut sur l'Entity

Lire l'entity existante pour vérifier si un champ de statut existe déjà.

Si absent, ajouter dans `src/Entity/{EntityClass}.php` :

```php
use Doctrine\ORM\Mapping as ORM;

// Dans la classe :
#[ORM\Column(length: 30)]
private string $status = 'draft';

public function getStatus(): string
{
    return $this->status;
}

public function setStatus(string $status): void
{
    $this->status = $status;
}
```

Puis mettre à jour le schéma :

```bash
php bin/console doctrine:migrations:diff
php bin/console doctrine:migrations:migrate
```

---

## Étape 4 — Créer la config YAML

Créer ou compléter `config/packages/workflow.yaml` :

```yaml
framework:
    workflows:
        order_process:  # snake_case du nom
            type: 'state_machine'  # ou 'workflow'
            audit_trail:
                enabled: true  # log chaque transition dans les logs Symfony
            marking_store:
                type: 'method'
                property: 'status'  # utilise getStatus()/setStatus()
            supports:
                - App\Entity\Order
            initial_marking: draft
            places:
                - draft
                - submitted
                - approved
                - rejected
                - shipped
                - cancelled
            transitions:
                submit:
                    from: draft
                    to:   submitted
                approve:
                    from: submitted
                    to:   approved
                reject:
                    from: submitted
                    to:   rejected
                ship:
                    from: approved
                    to:   shipped
                cancel:
                    from: [draft, submitted, approved]
                    to:   cancelled
```

**Personnaliser** les places et transitions selon le besoin métier.

---

## Étape 5 — Injecter et utiliser le Workflow dans un Service

Dans `src/Service/OrderService.php` (ou créer le service) :

```php
<?php
declare(strict_types=1);

namespace App\Service;

use App\Entity\Order;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Component\DependencyInjection\Attribute\Target;
use Symfony\Component\Workflow\WorkflowInterface;

final class OrderService
{
    public function __construct(
        private readonly EntityManagerInterface $entityManager,
        #[Target('orderProcess')]  // Nom camelCase du workflow Symfony
        private readonly WorkflowInterface $orderWorkflow,
    ) {}

    public function submit(Order $order): void
    {
        if (!$this->orderWorkflow->can($order, 'submit')) {
            throw new \LogicException(
                sprintf('Cannot submit order in status "%s".', $order->getStatus()),
            );
        }

        $this->orderWorkflow->apply($order, 'submit');
        $this->entityManager->flush();
    }

    public function approve(Order $order): void
    {
        if (!$this->orderWorkflow->can($order, 'approve')) {
            throw new \LogicException('Cannot approve this order.');
        }

        $this->orderWorkflow->apply($order, 'approve');
        $this->entityManager->flush();
    }

    /** @return string[] liste des transitions disponibles pour cet objet */
    public function getAvailableTransitions(Order $order): array
    {
        return array_map(
            fn ($t) => $t->getName(),
            $this->orderWorkflow->getEnabledTransitions($order),
        );
    }
}
```

---

## Étape 6 — Guard Listener (règles métier bloquantes)

Créer `src/EventListener/{WorkflowNamePascal}GuardListener.php` pour bloquer des transitions conditionnellement :

```php
<?php
declare(strict_types=1);

namespace App\EventListener;

use App\Entity\Order;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\Workflow\Event\GuardEvent;

final class OrderProcessGuardListener
{
    // Bloque la transition 'approve' si conditions non remplies
    #[AsEventListener(event: 'workflow.order_process.guard.approve')]
    public function onGuardApprove(GuardEvent $event): void
    {
        /** @var Order $order */
        $order = $event->getSubject();

        if ($order->getTotalAmount() > 10000 && !$order->hasManagerApproval()) {
            $event->setBlocked(true, 'Les commandes > 10 000 € nécessitent une approbation manager.');
        }
    }

    // Bloque l'expédition si l'adresse de livraison est manquante
    #[AsEventListener(event: 'workflow.order_process.guard.ship')]
    public function onGuardShip(GuardEvent $event): void
    {
        /** @var Order $order */
        $order = $event->getSubject();

        if ($order->getShippingAddress() === null) {
            $event->setBlocked(true, 'Adresse de livraison manquante.');
        }
    }
}
```

**Autres événements utiles :**
```
workflow.order_process.enter.approved    → déclenché quand approved est entré
workflow.order_process.leave.submitted   → déclenché quand submitted est quitté
workflow.order_process.transition.ship   → déclenché pendant la transition ship
workflow.order_process.completed.ship    → après la transition ship terminée
```

---

## Étape 7 — Visualiser le workflow

```bash
# Générer un graphe (nécessite graphviz)
php bin/console workflow:dump order_process | dot -Tpng -o /tmp/workflow.png

# Lister les workflows configurés
php bin/console debug:config framework workflows
```

---

## Résultat attendu

Rapporter :
- Fichiers créés ou modifiés
- Places et transitions configurées
- Champ de statut ajouté à l'entity (ou existant détecté)
- Commande de migration Doctrine si nécessaire
