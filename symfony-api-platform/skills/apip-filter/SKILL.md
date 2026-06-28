---
name: apip-filter
description: "Configure des filtres API Platform 3 sur une Resource : SearchFilter, OrderFilter, DateFilter via #[ApiFilter], ou crée un filtre custom héritant AbstractFilter."
argument-hint: "<EntityClass> [search|order|date|custom|all] — ex: Post search"
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep"]
---

# API Platform Filter

Configure des filtres de recherche, tri, date ou custom sur une Resource API Platform 3.

**Arguments:** "$ARGUMENTS"

Parse les arguments :
- `EntityClass` — nom de l'entity en PascalCase (ex. `Post`)
- `mode` — `search`, `order`, `date`, `custom`, ou `all` — défaut : `all`

---

## Étape 1 — Découverte

Lire l'entity pour connaître ses propriétés et les filtres déjà configurés :

```bash
grep -n '#\[ApiFilter\|#\[ApiResource' src/Entity/{EntityClass}.php 2>/dev/null
```

Vérifier les filtres custom existants :

```bash
find src/Filter -name '*.php' 2>/dev/null
```

---

## Étape 2 — Filtres built-in (SearchFilter, OrderFilter, DateFilter)

Ajouter les attributs `#[ApiFilter]` sur la classe Resource dans `src/Entity/{EntityClass}.php` :

```php
<?php
declare(strict_types=1);

namespace App\Entity;

use ApiPlatform\Doctrine\Orm\Filter\DateFilter;
use ApiPlatform\Doctrine\Orm\Filter\OrderFilter;
use ApiPlatform\Doctrine\Orm\Filter\SearchFilter;
use ApiPlatform\Metadata\ApiFilter;
use ApiPlatform\Metadata\ApiResource;
// ... autres imports

#[ApiResource(...)]
#[ApiFilter(
    SearchFilter::class,
    properties: [
        'title'         => 'partial',   // LIKE %value%     → ?title=symfony
        'status'        => 'exact',     // = value          → ?status=published
        'author.name'   => 'partial',   // relation         → ?author.name=john
        'tags.name'     => 'exact',     // collection       → ?tags.name=php
    ]
)]
#[ApiFilter(
    OrderFilter::class,
    properties: ['createdAt', 'title', 'status'],
    arguments: ['orderParameterName' => 'order']
    // Usage : ?order[createdAt]=desc&order[title]=asc
)]
#[ApiFilter(
    DateFilter::class,
    properties: ['createdAt', 'publishedAt']
    // Usage : ?createdAt[after]=2024-01-01&createdAt[before]=2024-12-31
    //         ?publishedAt[strictly_after]=2024-06-01
)]
class Post
{
    // ...
}
```

**Stratégies SearchFilter disponibles :**
```
'partial'    → LIKE %value%           (recherche dans le contenu)
'exact'      → = value                (correspondance exacte)
'start'      → LIKE value%            (commence par)
'end'        → LIKE %value            (termine par)
'word_start' → LIKE value% OR LIKE % value% (début de mot)
'ipartial'   → ILIKE %value%          (insensible à la casse)
'iexact'     → ILIKE value            (exact, insensible à la casse)
```

**DateFilter — opérateurs disponibles :**
```
?createdAt[after]=2024-01-01         → >= 2024-01-01
?createdAt[before]=2024-12-31        → <= 2024-12-31
?createdAt[strictly_after]=2024-01-01  → > 2024-01-01
?createdAt[strictly_before]=2024-12-31 → < 2024-12-31
```

---

## Étape 3 — RangeFilter et BooleanFilter (filtres additionnels)

```php
use ApiPlatform\Doctrine\Orm\Filter\BooleanFilter;
use ApiPlatform\Doctrine\Orm\Filter\RangeFilter;

#[ApiFilter(BooleanFilter::class, properties: ['isPublished'])]
// Usage : ?isPublished=true  ou  ?isPublished=false

#[ApiFilter(RangeFilter::class, properties: ['price', 'viewCount'])]
// Usage : ?price[gt]=10&price[lt]=100
//         ?viewCount[gte]=1000
```

---

## Étape 4 — Filtre custom (mode: custom)

Créer `src/Filter/{FilterName}Filter.php` pour une logique de filtrage non couverte par les filtres built-in :

```php
<?php
declare(strict_types=1);

namespace App\Filter;

use ApiPlatform\Doctrine\Orm\Filter\AbstractFilter;
use ApiPlatform\Doctrine\Orm\Util\QueryNameGeneratorInterface;
use ApiPlatform\Metadata\Operation;
use Doctrine\ORM\QueryBuilder;
use Doctrine\Persistence\ManagerRegistry;
use Psr\Log\LoggerInterface;
use Symfony\Component\Serializer\NameConverter\NameConverterInterface;

/**
 * Filters posts by published status.
 * Usage: ?published=true  or  ?published=false
 */
final class PublishedFilter extends AbstractFilter
{
    public function __construct(
        ManagerRegistry $managerRegistry,
        ?LoggerInterface $logger = null,
        ?array $properties = null,
        ?NameConverterInterface $nameConverter = null,
    ) {
        parent::__construct($managerRegistry, $logger, $properties, $nameConverter);
    }

    protected function filterProperty(
        string $property,
        mixed $value,
        QueryBuilder $queryBuilder,
        QueryNameGeneratorInterface $queryNameGenerator,
        string $resourceClass,
        ?Operation $operation = null,
        array $context = [],
    ): void {
        // Ne traiter que la propriété 'published' activée sur ce filtre
        if ($property !== 'published' || !$this->isPropertyEnabled($property, $resourceClass)) {
            return;
        }

        $alias     = $queryBuilder->getRootAliases()[0];
        $paramName = $queryNameGenerator->generateParameterName('publishedAt');

        if (filter_var($value, FILTER_VALIDATE_BOOLEAN)) {
            // Articles publiés : publishedAt n'est pas null et est dans le passé
            $queryBuilder
                ->andWhere(sprintf('%s.publishedAt IS NOT NULL', $alias))
                ->andWhere(sprintf('%s.publishedAt <= :%s', $alias, $paramName))
                ->setParameter($paramName, new \DateTimeImmutable());
        } else {
            // Articles non publiés
            $queryBuilder->andWhere(sprintf('%s.publishedAt IS NULL', $alias));
        }
    }

    /**
     * Describes the filter for OpenAPI documentation.
     */
    public function getDescription(string $resourceClass): array
    {
        if (!$this->properties) {
            return [];
        }

        $description = [];
        foreach ($this->properties as $property => $_) {
            $description[$property] = [
                'property' => $property,
                'type'     => 'bool',
                'required' => false,
                'openapi'  => [
                    'description' => 'Filter by published status. true = published, false = draft.',
                    'example'     => true,
                ],
            ];
        }

        return $description;
    }
}
```

Ajouter le filtre custom sur la Resource :

```php
use App\Filter\PublishedFilter;

#[ApiResource(...)]
#[ApiFilter(PublishedFilter::class, properties: ['published'])]
class Post { ... }
```

---

## Étape 5 — Vérifier dans l'OpenAPI

```bash
# Vérifier que les paramètres de filtre apparaissent dans la spec
php bin/console api:openapi:export | python3 -c "
import json, sys
spec = json.load(sys.stdin)
params = spec.get('paths', {}).get('/api/posts', {}).get('get', {}).get('parameters', [])
for p in params: print(p.get('name'))
"

# Alternative avec jq
php bin/console api:openapi:export | jq '[.paths."/api/posts".get.parameters[].name]'
```

---

## Résultat attendu

Rapporter :
- Filtres ajoutés sur la Resource (built-in et/ou custom)
- URLs d'exemple pour chaque filtre
- Classe de filtre custom créée si applicable
