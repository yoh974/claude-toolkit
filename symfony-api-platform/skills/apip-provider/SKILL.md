---
name: apip-provider
description: "Crée un ou plusieurs State Providers API Platform 3 : ItemProvider typé (Get), CollectionProvider (GetCollection), avec option de délégation au provider built-in pour pagination/filtres automatiques."
argument-hint: "<EntityClass> [item|collection|both] — ex: Post both"
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep"]
---

# API Platform State Provider

Crée des State Providers pour les opérations de lecture API Platform 3.

**Arguments:** "$ARGUMENTS"

Parse les arguments :
- `EntityClass` — nom de l'entity en PascalCase (ex. `Post`)
- `mode` — `item` (Get uniquement), `collection` (GetCollection uniquement), `both` (les deux) — défaut : `both`

---

## Étape 1 — Découverte

Vérifier les providers existants :

```bash
find src/State -name '{EntityClass}*Provider.php' 2>/dev/null
find src/State -name '*.php' 2>/dev/null | head -10
```

Lire l'entity pour connaître la structure :

```bash
find src/Entity -name "{EntityClass}.php" | head -1
```

Vérifier que le Repository existe :

```bash
find src/Repository -name "{EntityClass}Repository.php" | head -1
```

---

## Étape 2a — Créer l'ItemProvider (mode: item ou both)

Créer `src/State/{EntityClass}Provider.php` :

```php
<?php
declare(strict_types=1);

namespace App\State;

use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProviderInterface;
use App\Entity\Post;
use App\Repository\PostRepository;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

/**
 * @implements ProviderInterface<Post>
 */
final class PostProvider implements ProviderInterface
{
    public function __construct(private readonly PostRepository $repository) {}

    public function provide(Operation $operation, array $uriVariables = [], array $context = []): Post
    {
        $id = $uriVariables['id'] ?? null;

        $post = $this->repository->find($id);

        if ($post === null) {
            throw new NotFoundHttpException(
                sprintf('Post with id %s not found.', $id),
            );
        }

        return $post;
    }
}
```

---

## Étape 2b — Créer le CollectionProvider (mode: collection ou both)

**Option A — Provider custom simple** (pour des listes sans pagination/filtres complexes) :

Créer `src/State/{EntityClass}CollectionProvider.php` :

```php
<?php
declare(strict_types=1);

namespace App\State;

use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProviderInterface;
use App\Entity\Post;
use App\Repository\PostRepository;

/**
 * @implements ProviderInterface<iterable<Post>>
 */
final class PostCollectionProvider implements ProviderInterface
{
    public function __construct(private readonly PostRepository $repository) {}

    /**
     * @return Post[]
     */
    public function provide(Operation $operation, array $uriVariables = [], array $context = []): array
    {
        // Exemple de filtre métier custom avant de déléguer
        return $this->repository->findAllPublished();
    }
}
```

**Option B — Délégation au provider built-in** (recommandé pour pagination + filtres `#[ApiFilter]` automatiques) :

```php
<?php
declare(strict_types=1);

namespace App\State;

use ApiPlatform\Doctrine\Orm\State\CollectionProvider;
use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProviderInterface;
use App\Entity\Post;

/**
 * Delegates to the built-in Doctrine ORM provider to get pagination and #[ApiFilter] support.
 * Add custom pre/post-processing around the delegate call if needed.
 *
 * @implements ProviderInterface<Post>
 */
final class PostCollectionProvider implements ProviderInterface
{
    public function __construct(
        private readonly CollectionProvider $collectionProvider,
    ) {}

    public function provide(Operation $operation, array $uriVariables = [], array $context = []): object|array|null
    {
        // Pré-traitement possible ici (ex. modifier le $context)

        $result = $this->collectionProvider->provide($operation, $uriVariables, $context);

        // Post-traitement possible ici (ex. enrichir les items)

        return $result;
    }
}
```

**Choisir Option A** si : requête custom, filtre métier complexe, source de données non-Doctrine.
**Choisir Option B** si : collection Doctrine standard, filtres `#[ApiFilter]` sur la Resource.

---

## Étape 3 — Provider avec sécurité (filtrage par utilisateur)

Exemple de provider qui filtre les résultats selon l'utilisateur connecté :

```php
<?php
declare(strict_types=1);

namespace App\State;

use ApiPlatform\Metadata\Operation;
use ApiPlatform\State\ProviderInterface;
use App\Entity\Post;
use App\Repository\PostRepository;
use Symfony\Bundle\SecurityBundle\Security;

/**
 * @implements ProviderInterface<iterable<Post>>
 */
final class PostCollectionProvider implements ProviderInterface
{
    public function __construct(
        private readonly PostRepository $repository,
        private readonly Security $security,
    ) {}

    public function provide(Operation $operation, array $uriVariables = [], array $context = []): array
    {
        $user = $this->security->getUser();

        if ($this->security->isGranted('ROLE_ADMIN')) {
            return $this->repository->findAll();
        }

        // Utilisateur normal : uniquement ses propres articles + les publiés
        return $this->repository->findVisibleForUser($user);
    }
}
```

---

## Étape 4 — Vérifier l'enregistrement

```bash
php bin/console debug:container State 2>&1 | grep -i post

# Vérifier que les providers sont reconnus par API Platform
php bin/console debug:config api_platform 2>&1 | head -50
```

---

## Résultat attendu

Rapporter :
- Fichiers créés (ItemProvider et/ou CollectionProvider)
- Option choisie (custom vs délégation built-in) et raison
- Repository utilisé
