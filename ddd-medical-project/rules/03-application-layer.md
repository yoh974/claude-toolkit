---
description: Règles pour la couche Application (commands, queries, sagas, read models)
globs: src/**/application/**/*.ts
---

# Couche Application

## Commands

### Structure d'un dossier commande :

```
commands/
└── create-my/
    ├── create-my.command.ts
    ├── create-my.handler.ts
    └── create-my.handler.spec.ts
```

### Fichier Command :

```typescript
import { Command } from '@nestjs/cqrs';
import z from 'zod';

export const CreateMyCommandSchema = z.object({
  id: z.string().uuid().optional(),
  requiredField: z.string(),
  optionalField: z.string().optional(),
});

export type CreateMyCommandPayload = z.infer<typeof CreateMyCommandSchema>;
export type CreateMyCommandResult = { id: string };

export class CreateMyCommand extends Command<CreateMyCommandResult> {
  constructor(readonly payload: CreateMyCommandPayload) {
    super();
  }
}
```

### Fichier Handler :

```typescript
@CommandHandler(CreateMyCommand)
export class CreateMyHandler implements ICommandHandler<CreateMyCommand, CreateMyCommandResult> {
  private readonly logger = new Logger(CreateMyHandler.name);

  constructor(
    @Inject(MY_COMMAND_REPOSITORY)
    private readonly repository: MyCommandRepositoryPort,
  ) {}

  async execute(command: CreateMyCommand): Promise<CreateMyCommandResult> {
    // 1. Vérifier unicité si nécessaire
    // 2. Créer aggregate
    // 3. aggregate.apply(new MyCreated({...}))
    // 4. repository.persist(aggregate)
    // 5. Retourner { id }
  }
}
```

### Index des handlers :

```typescript
export const MyCommandHandlers = [CreateMyHandler, UpdateMyHandler];
```

## Queries

```typescript
import { Query } from '@core/cqrs'; // Pas @nestjs/cqrs

export class GetMyQuery extends Query<GetMyResponse> {
  constructor(readonly query: GetMyRequest) {
    super();
  }
}
```

- Utiliser `Query` de `@core/cqrs`
- Les réponses sont des plain objects
- Pagination avec limit/offset

## Read Models

- Modèles de lecture optimisés, indépendants du domaine et de l'infra
- Ce n'est PAS un Snapshot (domaine), ni un DTO (HTTP), ni une entité Prisma
- Flux : `DB → Adapter → Read Model → Controller → DTO HTTP`
- Le port de query retourne des Read Models

## Sagas

- Coordonnent plusieurs aggregates dans une transaction atomique
- Pattern Saga Orchestration avec steps (invoke + withCompensation)
- Compensations en ordre inverse (LIFO)
- Idempotence obligatoire (vérifier si existe déjà)
- État partagé entre steps via `params.state`

## Legacy Import Sagas (Anti-Corruption Layer)

- Saga = orchestration uniquement, AUCUNE transformation
- Step = appelle un mapper + émet une commande
- Mapper = pur, stateless, type d'entrée = contrat Kafka du core
- Utiliser `LegacySanitizer` pour les transformations communes
