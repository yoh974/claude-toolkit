# NestJS + DDD + CQRS + Event Sourcing

Microservices NestJS organisés en bounded contexts DDD, écrits en TypeScript, état reconstruit par replay d'événements.

## Couches & dépendances

```
Présentation → Application → Domaine ← Infrastructure
```

Le **Domaine** ne dépend de rien. Le module NestJS (Composition Root) dépend de toutes les couches et ne contient **aucune logique**, juste du wiring (providers, controllers, exports).

```
bounded-context/
├── bounded-context.module.ts   # à la racine, PAS dans application/
├── domain/{aggregates,events,exceptions,ports,value-objects}/
├── application/{commands,queries,read-models,sagas}/
├── infrastructure/{repositories,projections}/
└── presentation/{dto,*.controller.ts}/
```

## Aggregate (Event Sourcing)

Méthodes obligatoires : `getAggregateId()`, `getAggregateType()`, `takeSnapshot()`, `static fromSnapshot()`, `static create()`, `static fromId()` (état vide pour le replay).

```ts
async findById(id: MyId): Promise<MyAggregate | undefined> {
  const events = await this.eventStore.getAggregateEvents(id.value);
  if (events.length === 0) return undefined;
  const aggregate = MyAggregate.fromId(id);
  aggregate.loadFromHistory(events); // jamais de lecture directe sinon
  aggregate.commit();
  return aggregate;
}
```

Event handler dans l'aggregate : `on[EventName]Event(event) { this._state.field = event.payload.field; }`

Events nommés au **passé**, **sans** suffixe "Event" : `IdentityCreated` ✅, `IdentityCreatedEvent` ❌.

## Command / Handler

```
commands/create-my/create-my.{command,handler,handler.spec}.ts
```

```ts
@CommandHandler(CreateMyCommand)
export class CreateMyHandler implements ICommandHandler<CreateMyCommand, CreateMyCommandResult> {
  constructor(@Inject(MY_COMMAND_REPOSITORY) private readonly repository: MyCommandRepositoryPort) {}
  async execute(command: CreateMyCommand) {
    // 1. unicité  2. créer aggregate  3. aggregate.apply(new MyCreated({...}))
    // 4. repository.persist(aggregate)  5. return { id }
  }
}
```

## Query

Utiliser `Query` de `@core/cqrs` — **pas** `@nestjs/cqrs`. Retourne des Read Models (ni Snapshot domaine, ni DTO HTTP).

## Projections

Une projection **par événement**. Ordre des méthodes : `isEventAlreadyProjected → project → projectBatch → build* → map*`.
`build*` retourne le type **model** Prisma complet (pas `CreateInput`) car le driver Mongo natif ignore `@default()`.

```ts
@Projection(MyCreated)
export class MyCreatedProjectionHandler {
  async project(event, tx: PrismaTransaction) {
    await tx.myModel.create({ data: this.buildModel(event) }); // jamais this.prisma directement
  }
}
```

## Controller HTTP (passif)

Aucune logique métier, aucune transformation complexe, aucun `Logger`, aucun `try/catch`.

```ts
@Post()
@HttpCode(HttpStatus.CREATED)
create(@Body() dto: CreateMyRequestDto) {
  return this.commandBus.execute(new CreateMyCommand(dto));
}
```
Codes : 201 create, 200 read, 204 update/delete.

## Sagas

Steps avec `invoke` + `withCompensation`, compensations en ordre **inverse (LIFO)**, idempotence obligatoire (vérifier l'existence avant d'agir).

## Commandes

```bash
yarn build / start:dev / lint / format
yarn test / test:watch / test:coverage / test:e2e
npx jest path/to/file.spec.ts --no-coverage   # un seul fichier
npx tsc --noEmit                              # type-check seul
yarn prisma:generate / prisma:push / prisma:studio
```

## Pièges courants

- Oublier `loadFromHistory()` dans le Command Repository → lecture directe interdite en Event Sourcing
- `this.prisma` au lieu du paramètre `transaction` dans `project()` → casse l'atomicité
- Mettre de la logique ou un `try/catch` dans un controller HTTP
- Suffixer un event avec "Event" (`FooCreatedEvent` au lieu de `FooCreated`)
- Créer un `index.ts` barrel (interdit, sauf tableau de handlers consommé par le module)
