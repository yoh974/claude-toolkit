---
description: Règles pour la couche Domain (aggregates, events, exceptions, ports, value objects)
globs: src/**/domain/**/*.ts
---

# Couche Domain

## Aggregates

Tous les aggregates étendent `BaseAggregateRoot`.

### Méthodes obligatoires :

- `getAggregateId()` : retourne `this._state.id.value`
- `getAggregateType()` : retourne le nom du type
- `takeSnapshot()` : retourne un plain object (MyAggregateSnapshot)
- `static fromSnapshot(snapshot)` : reconstruit depuis un snapshot
- `static create()` : crée un nouvel aggregate avec un nouvel ID
- `static fromId(id)` : crée un aggregate vide pour l'Event Sourcing (état minimal, sera rempli par `loadFromHistory`)

### Event Handlers (Event Sourcing) :

```typescript
on[EventName]Event(event: MyEvent): void {
  this._state.field = event.payload.field;
}
```

### Types :

- `MyAggregateState` : avec Value Objects (pour usage interne)
- `MyAggregateSnapshot` : plain objects (pour sérialisation)

### Bonnes pratiques :

- Ne pas exposer `_state` directement
- Dates en strings ISO 8601
- Status : `'ACTIVE' | 'DELETED'`
- Toujours implémenter `fromId()` pour l'Event Sourcing

## Domain Events

```typescript
export type MyCreatedPayload = {
  myId: string;
  field: string;
  createdAt: string;
};

export class MyCreated extends BaseDomainEvent<MyCreatedPayload> {
  constructor(readonly payload: MyCreatedPayload) {
    super({ payload });
  }
}
```

- Nommés au **passé** : `MyCreated`, `MyUpdated`, `MyDeleted`
- **PAS** de suffixe "Event" : ✅ `CompanyCreated`, ❌ `CompanyCreatedEvent`
- Payload immuable, contient toutes les infos pour les projections
- IDs en strings, timestamps en ISO 8601

## Exceptions

```typescript
export class MyNotFoundException extends BaseDomainException {
  constructor(readonly myId: string) {
    super(`My entity with identifier "${myId}" does not exist`);
  }
}
```

- Format : `[Entity][Situation]Exception`
- Messages descriptifs avec identifiants entre guillemets
- Exposer les données via `readonly`

## Ports (Interfaces)

- Command : `MyCommandRepositoryPort extends CommandRepositoryPort<MyAggregate>`
- Query : `MyQueryRepositoryPort` retourne des Read Models (pas des Snapshots ni DTOs)
- Tokens : `export const MY_COMMAND_REPOSITORY = 'MY_COMMAND_REPOSITORY'`

## Value Objects

- IDs : `export class MyId extends UUIDv7Identifier {}`
- Complexes : `create()`, `fromSnapshot()`, `takeSnapshot()`, immuables
- Modifications retournent une nouvelle instance
