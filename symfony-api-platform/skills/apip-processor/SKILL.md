---
name: apip-processor
description: "Crée un State Processor API Platform 3 : persist (Post/Put), delete (Delete), ou async via Messenger. Un processor par comportement distinct."
argument-hint: "<EntityClass> [persist|delete|async] — ex: Post persist"
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep"]
---

# API Platform State Processor

Crée des State Processors pour les opérations d'écriture API Platform 3.

**Arguments:** "$ARGUMENTS"

Parse les arguments :
- `EntityClass` — nom de l'entity en PascalCase (ex. `Post`)
- `mode` — `persist` (Post/Put), `delete` (Delete), `async` (dispatch Messenger), ou `all` — défaut : `persist`

**Règle fondamentale** : un processor par comportement distinct. Pas de `if ($operation instanceof Delete)` dans un seul processor — deux classes séparées.

---

## Étape 1 — Découverte

Vérifier les processors existants :

```bash
find src/State -name '{EntityClass}*Processor.php' 2>/dev/null
find src/State -name '*.php' 2>/dev/null | head -10
```

Lire l'entity cible :

```bash
find src/Entity -name "{EntityClass}.php" | head -1
```

---

## Étape 2a — Persist Processor (mode: persist)

Créer `src/State/{EntityClass}Processor.php` pour les opérations `Post` et `Put` :

```php
<?php
declare(strict_types=1);

namespace App\State;

use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;
use App\Entity\Post;
use Doctrine\ORM\EntityManagerInterface;

/**
 * @implements ProcessorInterface<Post, Post>
 */
final class PostProcessor implements ProcessorInterface
{
    public function __construct(
        private readonly EntityManagerInterface $entityManager,
    ) {}

    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): Post
    {
        /** @var Post $data */

        // API Platform a déjà désérialisé le payload dans $data
        // et appliqué la validation — procéder directement à la persistance

        $this->entityManager->persist($data);
        $this->entityManager->flush();

        return $data;
    }
}
```

---

## Étape 2b — Delete Processor (mode: delete)

Créer `src/State/{EntityClass}DeleteProcessor.php` pour l'opération `Delete` :

```php
<?php
declare(strict_types=1);

namespace App\State;

use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;
use App\Entity\Post;
use Doctrine\ORM\EntityManagerInterface;

/**
 * @implements ProcessorInterface<Post, void>
 */
final class PostDeleteProcessor implements ProcessorInterface
{
    public function __construct(
        private readonly EntityManagerInterface $entityManager,
    ) {}

    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): void
    {
        /** @var Post $data */

        $this->entityManager->remove($data);
        $this->entityManager->flush();
    }
}
```

---

## Étape 2c — Async Processor via Messenger (mode: async)

Pour les opérations où l'écriture doit être asynchrone (files d'attente, longue durée) :

Créer d'abord le Command Message `src/Message/Create{EntityClass}Command.php` :

```php
<?php
declare(strict_types=1);

namespace App\Message;

final readonly class CreatePostCommand
{
    public function __construct(
        public readonly string $title,
        public readonly string $content,
        public readonly string $authorId,
    ) {}
}
```

Puis le Processor `src/State/{EntityClass}Processor.php` :

```php
<?php
declare(strict_types=1);

namespace App\State;

use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;
use App\Entity\Post;
use App\Message\CreatePostCommand;
use Symfony\Component\Messenger\MessageBusInterface;

/**
 * @implements ProcessorInterface<Post, Post>
 *
 * Dispatches a Messenger command for async processing.
 * Returns the input $data immediately (202 Accepted pattern).
 */
final class PostProcessor implements ProcessorInterface
{
    public function __construct(
        private readonly MessageBusInterface $messageBus,
    ) {}

    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): Post
    {
        /** @var Post $data */

        $this->messageBus->dispatch(new CreatePostCommand(
            title: $data->getTitle(),
            content: $data->getContent(),
            authorId: $data->getAuthor()?->getId() ?? '',
        ));

        // Retourner l'objet non-persisté — le handler Messenger le persistera
        return $data;
    }
}
```

---

## Étape 3 — Processor avec logique métier avant persistance

Exemple d'un processor qui applique des règles métier :

```php
<?php
declare(strict_types=1);

namespace App\State;

use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProcessorInterface;
use App\Entity\Post;
use App\Service\SlugGenerator;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Bundle\SecurityBundle\Security;

/**
 * @implements ProcessorInterface<Post, Post>
 */
final class PostProcessor implements ProcessorInterface
{
    public function __construct(
        private readonly EntityManagerInterface $entityManager,
        private readonly SlugGenerator $slugGenerator,
        private readonly Security $security,
    ) {}

    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): Post
    {
        /** @var Post $data */

        // Assigner l'auteur automatiquement si c'est une création
        if ($data->getId() === null) {
            $user = $this->security->getUser();
            if ($user !== null) {
                $data->setAuthor($user);
            }
        }

        // Générer un slug depuis le titre si absent
        if (empty($data->getSlug())) {
            $data->setSlug($this->slugGenerator->generate($data->getTitle()));
        }

        $this->entityManager->persist($data);
        $this->entityManager->flush();

        return $data;
    }
}
```

---

## Étape 4 — Vérifier l'enregistrement

```bash
php bin/console debug:container State 2>&1 | grep -i post

# Vérifier dans la config API Platform
php bin/console api:openapi:export 2>&1 | head -50
```

---

## Résultat attendu

Rapporter :
- Fichiers créés (PersistProcessor, DeleteProcessor, et/ou commande Messenger)
- Mode utilisé (synchrone Doctrine vs async Messenger)
- Logique métier ajoutée au processor si applicable
