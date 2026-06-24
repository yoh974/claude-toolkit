---
description: Standards de code, nommage, style et imports
globs: src/**/*.ts
---

# Standards de Code

## Nommage de Fichiers (kebab-case + suffixe)

| Type          | Suffixe                  | Exemple                                          |
| ------------- | ------------------------ | ------------------------------------------------ |
| Aggregate     | `.aggregate.ts`          | `identity.aggregate.ts`                          |
| Event         | `.event.ts`              | `identity-created.event.ts`                      |
| Exception     | `.exception.ts`          | `identity-not-found.exception.ts`                |
| Value Object  | `.vo.ts`                 | `identity-id.vo.ts`                              |
| Port          | `.port.ts`               | `identity-command-repository.port.ts`            |
| Adapter       | `.adapter.ts`            | `prisma-identity-command-repository.adapter.ts`  |
| Command       | `.command.ts`            | `create-identity.command.ts`                     |
| Query         | `.query.ts`              | `get-identity.query.ts`                          |
| Handler       | `.handler.ts`            | `create-identity.handler.ts`                     |
| Projection    | `.projection.handler.ts` | `identity-created.projection.handler.ts`         |
| DTO           | `.dto.ts`                | `create-identity-request.dto.ts`                 |
| Controller    | `.controller.ts`         | `identity-http.controller.ts`                    |
| Interceptor   | `.interceptor.ts`        | `identity-http-domain-exceptions.interceptor.ts` |
| Module        | `.module.ts`             | `identity.module.ts`                             |
| Test unitaire | `.spec.ts`               | `create-identity.handler.spec.ts`                |
| Test E2E      | `.e2e-spec.ts`           | `create-identity.e2e-spec.ts`                    |
| Read Model    | `.read-model.ts`         | `identity.read-model.ts`                         |

## Nommage Classes/Variables

- Classes : `PascalCase`
- Variables/méthodes : `camelCase`
- Constantes/tokens injection : `SCREAMING_SNAKE_CASE`
- Events nommés au passé : `IdentityCreated` (pas `CreateIdentity`)
- Events SANS suffixe "Event" : `CompanyCreated` (pas `CompanyCreatedEvent`)

## Barrel Files (index.ts)

**NE PAS créer de fichiers barrel (index.ts)** sauf pour les tableaux de handlers/providers consommés par les modules.

```typescript
// ❌ MAUVAIS
import { ExposureSituationId, ExposureId } from '../value-objects';

// ✅ BON
import { ExposureSituationId } from '../value-objects/exposure-situation-id.vo';
import { ExposureId } from '../value-objects/exposure-id.vo';
```

## Style de Code

- Prettier : 2 espaces, single quotes, trailing commas ES5
- Types explicites obligatoires
- async/await plutôt que promise chains
- Union types plutôt qu'enums pour les statuts : `'ACTIVE' | 'DELETED'`
- `readonly` pour l'immutabilité
- Lancer des exceptions domaine, ne pas catcher dans les handlers

## Ordre des Imports

1. Externes (`@nestjs/*`, etc.)
2. `@core/*`
3. `@shared/*`
4. Alias cross-module (`@identity/*`, `@health/*`, etc.)
5. Relatifs (même module)

## Imports & Alias

- **Cross-module** : utiliser les alias (`@identity/domain/...`)
- **Même module** : utiliser les chemins relatifs (`../../../domain/...`)
