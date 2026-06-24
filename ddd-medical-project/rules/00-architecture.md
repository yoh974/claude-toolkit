---
description: Architecture DDD, CQRS, Event Sourcing et structure du projet
globs: src/**/*
---

# Architecture DDD + CQRS + Event Sourcing

## Structure des Modules

Chaque bounded context suit cette structure :

```
bounded-context/
├── bounded-context.module.ts  # Composition Root (À LA RACINE, pas dans application/)
├── application/
│   ├── commands/              # Commandes CQRS (écriture)
│   ├── queries/               # Queries CQRS (lecture)
│   ├── read-models/           # Read Models (vues optimisées lecture)
│   └── sagas/                 # Sagas / Process Managers
├── domain/
│   ├── aggregates/            # Extend BaseAggregateRoot
│   ├── events/                # Extend BaseDomainEvent
│   ├── exceptions/            # Extend BaseDomainException
│   ├── ports/                 # Interfaces repositories (command + query)
│   └── value-objects/         # Value Objects immuables
├── infrastructure/
│   ├── repositories/          # Prisma adapters implémentant les ports
│   └── projections/           # Event handlers pour read models
└── presentation/
    ├── dto/                   # Request/Response DTOs
    │   ├── request/
    │   └── response/
    ├── [module]-http.controller.ts
    └── [module]-http-domain-exceptions.interceptor.ts
```

## Dépendances entre Couches

```
Présentation → Application → Domaine ← Infrastructure
```

- Le **Domaine** ne dépend d'aucune autre couche
- Le **Composition Root** (module) dépend de TOUTES les couches (c'est sa raison d'être)
- Le module n'est PAS une couche DDD

## Pans Fonctionnels

Le code est organisé en pans fonctionnels :

- Simples (1 BC) : `identity/`, `documents/`, `affiliation/`
- Composites (N BC) : `work/` (exposure-situation...), `health/` (clinical-health, mental-health...)

Un BC ne doit PAS dépendre d'un autre BC du même pan fonctionnel. Les dépendances inter-BC passent par des événements de domaine.

## CQRS

- **Commands** : modifient l'état, retournent void ou un identifiant
- **Queries** : ne modifient pas l'état, lisent les read models (projections)

## Event Sourcing

L'état d'un Aggregate est reconstruit en rejouant tous ses événements :

```typescript
async findById(id: MyId): Promise<MyAggregate | undefined> {
  const events = await this.eventStore.getAggregateEvents(id.value);
  if (events.length === 0) return undefined;
  const aggregate = MyAggregate.fromId(id);
  aggregate.loadFromHistory(events);
  aggregate.commit();
  return aggregate;
}
```

Les Queries ne reconstruisent PAS les aggregates — elles lisent les projections.

## Composition Root (Module NestJS)

```typescript
@Module({
  controllers: [MyHttpController],
  providers: [
    ...MyCommandHandlers,
    ...MyQueryHandlers,
    ...MyProjectionHandlers,
    { provide: MY_COMMAND_REPOSITORY, useClass: PrismaMyCommandRepositoryAdapter },
    { provide: MY_QUERY_REPOSITORY, useClass: PrismaMyQueryRepositoryAdapter },
  ],
  exports: [MY_COMMAND_REPOSITORY, MY_QUERY_REPOSITORY],
})
export class MyModule {}
```

Le module ne contient JAMAIS de logique — uniquement du wiring.
