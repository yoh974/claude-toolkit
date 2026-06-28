---
name: apip-resource
description: "Ajoute ou modifie un #[ApiResource] API Platform 3 sur une Entity : opérations explicites, groupes de sérialisation, providers/processors, pagination, #[ApiProperty]."
argument-hint: "<EntityClass> — ex: Post"
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep"]
---

# API Platform Resource

Configure `#[ApiResource]` sur une Entity Doctrine avec opérations, sérialisation et sécurité.

**Arguments:** "$ARGUMENTS"

Parse les arguments :
- `EntityClass` — nom de l'entity en PascalCase (ex. `Post`)

Dériver :
- `entityLower` — minuscules (ex. `post`)
- `groupRead` — `{entityLower}:read`
- `groupWrite` — `{entityLower}:write`

---

## Étape 1 — Découverte

Lire les resources existantes pour respecter les conventions du projet :

```bash
grep -r '#\[ApiResource' src/ --include='*.php' -l 2>/dev/null | head -5
```

Lire une resource existante pour vérifier les patterns de groupes et d'opérations utilisés.

Trouver l'entity cible :

```bash
find src/Entity -name "{EntityClass}.php" 2>/dev/null | head -3
```

Lire l'entity pour connaître ses propriétés existantes.

---

## Étape 2 — Vérifier les dépendances

Vérifier si les Providers et Processors existent déjà :

```bash
find src/State -name "{EntityClass}*.php" 2>/dev/null
```

Note : les Providers/Processors seront créés avec `/apip-provider` et `/apip-processor` si absents.

---

## Étape 3 — Ajouter `#[ApiResource]` sur l'Entity

Modifier `src/Entity/{EntityClass}.php` :

```php
<?php
declare(strict_types=1);

namespace App\Entity;

use ApiPlatform\Metadata\ApiFilter;
use ApiPlatform\Metadata\ApiProperty;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Delete;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\Post;
use ApiPlatform\Metadata\Put;
use App\State\PostCollectionProvider;
use App\State\PostDeleteProcessor;
use App\State\PostProcessor;
use App\State\PostProvider;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Serializer\Attribute\Groups;
use Symfony\Component\Validator\Constraints as Assert;

#[ORM\Entity(repositoryClass: PostRepository::class)]
#[ORM\Table(name: 'posts')]
#[ApiResource(
    shortName: 'Post',
    operations: [
        new GetCollection(provider: PostCollectionProvider::class),
        new Get(provider: PostProvider::class),
        new Post(
            processor: PostProcessor::class,
            validationContext: ['groups' => ['Default', 'post:create']],
        ),
        new Put(
            provider: PostProvider::class,
            processor: PostProcessor::class,
        ),
        new Delete(
            provider: PostProvider::class,
            processor: PostDeleteProcessor::class,
        ),
    ],
    normalizationContext: ['groups' => ['post:read']],
    denormalizationContext: ['groups' => ['post:write']],
    paginationEnabled: true,
    paginationItemsPerPage: 30,
)]
class Post
{
    #[ORM\Id, ORM\GeneratedValue, ORM\Column]
    #[Groups(['post:read'])]
    #[ApiProperty(identifier: true, description: 'The unique post identifier')]
    private ?int $id = null;

    #[ORM\Column(length: 255)]
    #[Groups(['post:read', 'post:write'])]
    #[Assert\NotBlank(groups: ['Default', 'post:create'])]
    #[Assert\Length(max: 255)]
    #[ApiProperty(description: 'Post title', example: 'My first post')]
    private string $title = '';

    #[ORM\Column(type: 'text')]
    #[Groups(['post:read', 'post:write'])]
    #[Assert\NotBlank]
    #[ApiProperty(description: 'Post content in markdown', example: '# Hello\n\nContent here.')]
    private string $content = '';

    #[ORM\Column]
    #[Groups(['post:read'])]
    #[ApiProperty(description: 'Creation date (ISO 8601)', example: '2024-01-15T10:30:00+00:00')]
    private \DateTimeImmutable $createdAt;

    #[ORM\Column(nullable: true)]
    #[Groups(['post:read'])]
    private ?\DateTimeImmutable $publishedAt = null;

    public function __construct()
    {
        $this->createdAt = new \DateTimeImmutable();
    }

    public function getId(): ?int { return $this->id; }

    public function getTitle(): string { return $this->title; }
    public function setTitle(string $title): static { $this->title = $title; return $this; }

    public function getContent(): string { return $this->content; }
    public function setContent(string $content): static { $this->content = $content; return $this; }

    public function getCreatedAt(): \DateTimeImmutable { return $this->createdAt; }
    public function getPublishedAt(): ?\DateTimeImmutable { return $this->publishedAt; }
    public function isPublished(): bool { return $this->publishedAt !== null; }

    public function publish(): void
    {
        $this->publishedAt = new \DateTimeImmutable();
    }
}
```

**Règles sur les groupes :**
- `post:read` — tout ce qui est lu par le client
- `post:write` — tout ce qui est écrit par le client
- Les propriétés système (id, createdAt) : `post:read` uniquement
- Ne jamais mettre `post:write` sur `id` ou les dates générées

---

## Étape 4 — Vérifier la configuration OpenAPI

```bash
php bin/console api:openapi:export 2>&1 | head -100
```

Si des erreurs apparaissent (Provider/Processor manquants) : les créer avec `/apip-provider` et `/apip-processor`.

```bash
# Lister les routes générées
php bin/console debug:router | grep '/api/posts'
```

---

## Étape 5 — Sécurité sur les opérations (optionnel)

Ajouter `security:` sur les opérations qui le nécessitent :

```php
#[ApiResource(
    operations: [
        new GetCollection(),                                           // public
        new Get(security: "is_granted('POST_VIEW', object)"),         // Voter
        new Post(security: "is_granted('ROLE_USER')"),                // authentifié
        new Put(security: "is_granted('POST_EDIT', object)"),         // Voter
        new Delete(security: "is_granted('ROLE_ADMIN')"),             // admin uniquement
    ],
)]
```

---

## Résultat attendu

Rapporter :
- Attributs `#[ApiResource]` ajoutés
- Groupes de sérialisation configurés
- Opérations générées avec leurs routes
- Providers/Processors manquants (à créer avec `/apip-provider` et `/apip-processor`)
