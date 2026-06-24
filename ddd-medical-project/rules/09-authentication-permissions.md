---
description: Règles pour l'authentification, les permissions et le contexte utilisateur
globs: src/**/presentation/**/*.controller.ts
---

# Authentication & Permissions

## Décorateurs

Deux niveaux d'autorisation :

### `@Authenticated()` — Route nécessitant une authentification

```typescript
@Authenticated()
@Get('my-route')
findOne() { ... }
```

### `@Permissions(resource, action)` — Contrôle fin par ressource

```typescript
import { Permissions } from '@core/authentication/permissions/permissions.decorator';
import { PermissionResource } from '@core/authentication/permissions/permission-resource';

@Permissions(PermissionResource.IDENTITE, 'READ')
@Get(':id')
findOne() { ... }

@Permissions(PermissionResource.IDENTITE, 'CREATE')
@Post()
create() { ... }
```

Les deux décorateurs appliquent automatiquement `@UseGuards()` et `@ApiBearerAuth()`.

## Permission Resources

Constantes organisées par domaine dans `@core/authentication/permissions/permission-resource.ts` :

- IDENTITÉ : identity, addresses, emails, phones
- TRAVAIL : work situations, activities, exposure
- SOCIAL : life situations, absences, incapacities
- SANTÉ : health, clinical, mental, lifestyle

Actions : `CREATE`, `READ`, `UPDATE`, `DELETE`, `COMMENT`

## Guards

- Bypass automatique en `NODE_ENV === 'local'`
- Support dual transport : HTTP headers ET Kafka metadata
- Le user est injecté dans `ContextService` pour accès downstream

## Règles

- Toujours utiliser `@Permissions()` (pas de `@UseGuards()` custom pour l'auth)
- Les vérifications fines par item se font dans les command handlers via les règles domaine
