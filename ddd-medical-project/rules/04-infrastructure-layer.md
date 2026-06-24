---
description: Règles pour la couche Infrastructure (repositories, projections, Prisma)
globs: src/**/infrastructure/**/*.ts
---

# Couche Infrastructure

## Command Repository (Event Sourcing)

```typescript
@Injectable()
export class PrismaMyCommandRepositoryAdapter
  extends BasePrismaCommandRepository<MyAggregate>
  implements MyCommandRepositoryPort
{
  async findById(id: MyId): Promise<MyAggregate | undefined> {
    const events = await this.eventStore.getAggregateEvents(id.value);
    if (events.length === 0) return undefined;
    const aggregate = MyAggregate.fromId(id);
    aggregate.loadFromHistory(events);
    aggregate.commit();
    return aggregate;
  }
}
```

Toujours utiliser `loadFromHistory()`, jamais de lecture directe dans les Command Repositories.

## In-Memory Repository (Tests)

```typescript
export class InMemoryMyCommandRepositoryAdapter implements MyCommandRepositoryPort {
  private readonly entities = new Map<string, MyAggregate>();
  findById(id: MyId) {
    return Promise.resolve(this.entities.get(id.value));
  }
  persist(aggregate: MyAggregate) {
    this.entities.set(aggregate.getAggregateId(), aggregate);
    return Promise.resolve();
  }
  add(aggregate: MyAggregate) {
    this.entities.set(aggregate.getAggregateId(), aggregate);
  }
}
```

## Query Repository

- Retourne des **Read Models** (pas Snapshot, pas DTO, pas Prisma)
- Mapping DB → Read Model dans `toReadModel()` privé
- Pour objets complexes : mappers dédiés dans `infrastructure/repositories/query/mappers/`

## Projections

### Organisation :

```
infrastructure/projections/
├── index.ts                                    # Export: MyProjectionHandlers = [...]
├── my-created.projection.handler.ts
├── my-updated.projection.handler.ts
└── my-deleted.projection.handler.ts
```

### Ordre des méthodes :

1. `isEventAlreadyProjected`
2. `project` (transaction Prisma)
3. `projectBatch` (MongoBulkWriterService)
4. `build*` (retourne type Prisma **model**, pas CreateInput)
5. `map*` (retourne type Prisma **embedded**, pas CreateInput)

### Règles critiques :

- **Une projection par événement**
- **Utiliser la transaction** dans `project()`, jamais `this.prisma` directement
- `build*` retourne le type model complet (MongoDB driver ne gère pas `@default()`)
- `build*` est partagé entre `project()` et `projectBatch()`

## Conversions de Types

```typescript
// Prisma Date → string
createdAt: record.createdAt.toISOString(),
// string → Prisma Date
createdAt: new Date(payload.createdAt),
// Prisma null → undefined
optionalField: record.optionalField ?? undefined,
```
