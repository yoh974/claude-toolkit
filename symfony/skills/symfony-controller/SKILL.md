---
name: symfony-controller
description: "Crée ou modifie un Controller Symfony 7 : AbstractController, routes en attributs PHP, DTOs avec #[MapRequestPayload], #[IsGranted], REST complet CRUD."
argument-hint: "<ControllerName> [EntityClass] — ex: PostController Post"
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep"]
---

# Symfony Controller

Crée un Controller REST Symfony 7 avec DTOs, validation, sécurité et délégation aux Services.

**Arguments:** "$ARGUMENTS"

Parse les arguments :
- `ControllerName` — nom du controller en PascalCase (ex. `PostController`)
- `EntityClass` — classe de l'entité (ex. `Post`) — optionnel, déduit du nom si absent

Dériver :
- `EntityCamel` — camelCase de l'entity (ex. `post`)
- `routePrefix` — kebab-case (ex. `post`)

---

## Étape 1 — Découverte

Lire les controllers existants pour respecter les conventions du projet :

```bash
find src/Controller -name '*.php' | head -5
```

Vérifier si le Voter existe pour l'entity :

```bash
find src/Security/Voter -name "{EntityClass}Voter.php" 2>/dev/null
```

Vérifier si le Service existe :

```bash
find src/Service -name "{EntityClass}Service.php" 2>/dev/null
```

---

## Étape 2 — Créer les DTOs

Créer `src/DTO/Create{EntityClass}Dto.php` :

```php
<?php
declare(strict_types=1);

namespace App\DTO;

use Symfony\Component\Validator\Constraints as Assert;

final class CreatePostDto
{
    public function __construct(
        #[Assert\NotBlank]
        #[Assert\Length(min: 3, max: 255)]
        public readonly string $title = '',

        #[Assert\NotBlank]
        public readonly string $content = '',
    ) {}
}
```

Créer `src/DTO/Update{EntityClass}Dto.php` (champs optionnels avec `?`) :

```php
<?php
declare(strict_types=1);

namespace App\DTO;

use Symfony\Component\Validator\Constraints as Assert;

final class UpdatePostDto
{
    public function __construct(
        #[Assert\Length(min: 3, max: 255)]
        public readonly ?string $title = null,

        public readonly ?string $content = null,
    ) {}
}
```

---

## Étape 3 — Créer le Controller

Créer `src/Controller/{ControllerName}.php` :

```php
<?php
declare(strict_types=1);

namespace App\Controller;

use App\DTO\CreatePostDto;
use App\DTO\UpdatePostDto;
use App\Entity\Post;
use App\Repository\PostRepository;
use App\Security\Voter\PostVoter;
use App\Service\PostService;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Attribute\MapRequestPayload;
use Symfony\Component\Routing\Attribute\Route;
use Symfony\Component\Security\Http\Attribute\IsGranted;

#[Route('/api/posts', name: 'api_post_')]
final class PostController extends AbstractController
{
    public function __construct(private readonly PostService $postService) {}

    #[Route('', name: 'list', methods: ['GET'])]
    public function list(PostRepository $repository): JsonResponse
    {
        return $this->json($repository->findAll());
    }

    #[Route('/{id}', name: 'show', requirements: ['id' => '\d+'], methods: ['GET'])]
    public function show(Post $post): JsonResponse
    {
        $this->denyAccessUnlessGranted(PostVoter::VIEW, $post);

        return $this->json($post);
    }

    #[Route('', name: 'create', methods: ['POST'])]
    #[IsGranted('ROLE_USER')]
    public function create(#[MapRequestPayload] CreatePostDto $dto): JsonResponse
    {
        $post = $this->postService->create($dto);

        return $this->json($post, Response::HTTP_CREATED);
    }

    #[Route('/{id}', name: 'update', requirements: ['id' => '\d+'], methods: ['PUT', 'PATCH'])]
    public function update(Post $post, #[MapRequestPayload] UpdatePostDto $dto): JsonResponse
    {
        $this->denyAccessUnlessGranted(PostVoter::EDIT, $post);
        $this->postService->update($post, $dto);

        return $this->json($post);
    }

    #[Route('/{id}', name: 'delete', requirements: ['id' => '\d+'], methods: ['DELETE'])]
    public function delete(Post $post): Response
    {
        $this->denyAccessUnlessGranted(PostVoter::DELETE, $post);
        $this->postService->delete($post);

        return new Response(status: Response::HTTP_NO_CONTENT);
    }
}
```

**Notes sur les patterns utilisés :**
- `#[MapRequestPayload]` (Symfony 6.3+) : désérialise le corps JSON et valide les `#[Assert\]` automatiquement — renvoie 422 si invalide
- `requirements: ['id' => '\d+']` : évite les ambiguïtés de routing avec d'autres routes
- ParamConverter implicite : Symfony résout `Post $post` depuis l'`id` de la route via Doctrine
- `final class` : empêche l'héritage non intentionnel

---

## Étape 4 — Vérifier les routes

```bash
php bin/console debug:router | grep api_post
```

---

## Étape 5 — Si le Service n'existe pas

Créer un Service minimal `src/Service/{EntityClass}Service.php` pour que le controller compile :

```php
<?php
declare(strict_types=1);

namespace App\Service;

use App\DTO\CreatePostDto;
use App\DTO\UpdatePostDto;
use App\Entity\Post;
use App\Repository\PostRepository;
use Doctrine\ORM\EntityManagerInterface;

final class PostService
{
    public function __construct(
        private readonly EntityManagerInterface $entityManager,
        private readonly PostRepository $repository,
    ) {}

    public function create(CreatePostDto $dto): Post
    {
        $post = new Post();
        $post->setTitle($dto->title);
        $post->setContent($dto->content);

        $this->entityManager->persist($post);
        $this->entityManager->flush();

        return $post;
    }

    public function update(Post $post, UpdatePostDto $dto): void
    {
        if ($dto->title !== null) {
            $post->setTitle($dto->title);
        }
        if ($dto->content !== null) {
            $post->setContent($dto->content);
        }

        $this->entityManager->flush();
    }

    public function delete(Post $post): void
    {
        $this->entityManager->remove($post);
        $this->entityManager->flush();
    }
}
```

---

## Résultat attendu

Rapporter :
- Fichiers créés (DTOs, Controller, Service si créé)
- Routes générées
- Voter et Service détectés ou manquants
