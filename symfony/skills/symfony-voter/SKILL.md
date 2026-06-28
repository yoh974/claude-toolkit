---
name: symfony-voter
description: "Crée ou modifie un Symfony Security Voter : AbstractVoter, constantes d'attributs typées, supports(), voteOnAttribute() avec match exhaustif, injection de Security."
argument-hint: "<VoterName> <EntityClass> — ex: PostVoter Post"
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep"]
---

# Symfony Voter

Crée un Security Voter Symfony 7 complet suivant les bonnes pratiques.

**Arguments:** "$ARGUMENTS"

Parse les arguments :
- `VoterName` — nom du voter en PascalCase (ex. `PostVoter`)
- `EntityClass` — classe de l'entité en PascalCase (ex. `Post`)

Dériver :
- `VOTER_PREFIX` — SCREAMING_SNAKE_CASE de l'entité (ex. `POST`)
- Namespace de l'entity : à découvrir à l'étape 1

---

## Étape 1 — Découverte

Chercher les voters existants pour respecter le namespace et les conventions du projet :

```bash
find src/Security/Voter -name '*Voter.php' 2>/dev/null | head -5
```

Si trouvé, lire un voter existant pour vérifier le namespace exact et les imports.

Chercher aussi l'entity cible :

```bash
find src/Entity -name "{EntityClass}.php" 2>/dev/null | head -3
grep -r "class {EntityClass}" src/ --include="*.php" -l 2>/dev/null | head -3
```

---

## Étape 2 — Créer le Voter

Créer `src/Security/Voter/{VoterName}.php` :

```php
<?php
declare(strict_types=1);

namespace App\Security\Voter;

use App\Entity\Post;
use App\Entity\User;
use Symfony\Bundle\SecurityBundle\Security;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\Authorization\Voter\Voter;

final class PostVoter extends Voter
{
    public const string VIEW   = 'POST_VIEW';
    public const string EDIT   = 'POST_EDIT';
    public const string DELETE = 'POST_DELETE';

    public function __construct(private readonly Security $security) {}

    protected function supports(string $attribute, mixed $subject): bool
    {
        return in_array($attribute, [self::VIEW, self::EDIT, self::DELETE], true)
            && $subject instanceof Post;
    }

    protected function voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token): bool
    {
        $user = $token->getUser();
        if (!$user instanceof User) {
            return false;
        }

        /** @var Post $post */
        $post = $subject;

        return match ($attribute) {
            self::VIEW   => $this->canView($post, $user),
            self::EDIT   => $this->canEdit($post, $user),
            self::DELETE => $this->canDelete($post, $user),
            default      => false,
        };
    }

    private function canView(Post $post, User $user): bool
    {
        return $post->isPublished() || $this->canEdit($post, $user);
    }

    private function canEdit(Post $post, User $user): bool
    {
        return $this->security->isGranted('ROLE_ADMIN') || $post->getAuthor() === $user;
    }

    private function canDelete(Post $post, User $user): bool
    {
        return $this->security->isGranted('ROLE_ADMIN');
    }
}
```

**Adapter :**
- Remplacer `Post` / `User` par les classes réelles
- Adapter le préfixe (`POST_`) selon l'entity
- Ajuster les méthodes privées aux règles métier du projet

---

## Étape 3 — Vérifier l'autowiring

Symfony 7 avec `autoconfigure: true` enregistre automatiquement les voters — pas de config YAML nécessaire :

```bash
php bin/console debug:autowiring --all 2>&1 | grep -i voter
```

---

## Étape 4 — Exemples d'utilisation

**Dans un Controller :**
```php
// Bloquer si non autorisé (lève 403)
$this->denyAccessUnlessGranted(PostVoter::EDIT, $post);

// Vérifier sans bloquer
if ($this->isGranted(PostVoter::VIEW, $post)) { ... }
```

**Dans un template Twig :**
```twig
{% if is_granted('POST_EDIT', post) %}
    <a href="{{ path('api_post_update', {id: post.id}) }}">Modifier</a>
{% endif %}
```

**Dans un Service (injecter `Security`) :**
```php
use Symfony\Bundle\SecurityBundle\Security;

public function __construct(private readonly Security $security) {}

public function canUserEdit(Post $post): bool
{
    return $this->security->isGranted(PostVoter::EDIT, $post);
}
```

---

## Résultat attendu

Rapporter :
- Fichier créé avec son chemin
- Constantes d'attributs générées
- Toute incohérence de namespace détectée
