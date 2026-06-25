---
name: ts-typing-nestjs
description: "Guide de typage TypeScript niveau expert pour NestJS (stack DDD/CQRS/Event Sourcing). À appliquer quand on écrit ou refactore du TypeScript dans un projet NestJS (DTOs, commands, queries, handlers, domain events, controllers, consumers Kafka) et qu'on veut un typage robuste sans any ni cast."
---

# Typage TypeScript expert — NestJS (DDD / CQRS / Event Sourcing)

Quand tu écris ou refactores du TypeScript dans un projet NestJS, applique ces techniques.
Objectif : **zéro `any`, zéro cast (`as`) non justifié, zéro duplication de types**. Le domaine
doit être impossible à mal utiliser au niveau du type.

Pour chaque technique : un ❌ (typage faible) → ✅ (typage expert). Code court, orienté DDD/CQRS.

---

## 1. Utility types — dériver DTOs et payloads

But : un DTO de création/mise à jour se dérive de l'entité au lieu d'être réécrit.

```typescript
interface Individual {
  id: string;
  firstName: string;
  lastName: string;
  createdAt: Date;
}

// ✅ dérivés — restent synchro avec l'entité
type CreateIndividualDto = Omit<Individual, 'id' | 'createdAt'>;
type UpdateIndividualDto = Partial<Omit<Individual, 'id' | 'createdAt'>>;
type IndividualSummary = Pick<Individual, 'id' | 'lastName'>;

// Record pour un mapping statut → handler, clés exhaustives
type StatusHandlers = Record<Individual['id'] extends string ? string : never, () => void>;
```

## 2. Mapped types — payload depuis l'entité de domaine

But : générer un type structurel à partir des clés d'un autre (ex. version « sérialisée » pour Kafka).

```typescript
// transforme chaque Date en string ISO pour le transport
type Serialized<T> = {
  [K in keyof T]: T[K] extends Date ? string : T[K];
};

type IndividualEventPayload = Serialized<Individual>;
// createdAt devient string, le reste inchangé
```

## 3. Conditional types — résoudre le résultat d'un handler

But : lier le type de retour au type de commande/query traitée.

```typescript
interface ICommand<TResult = void> {
  readonly __result?: TResult;
}

// extrait le résultat déclaré par la commande
type ResultOf<C> = C extends ICommand<infer R> ? R : never;

class CreateIndividualCommand implements ICommand<string> {
  constructor(public readonly dto: CreateIndividualDto) {}
}

type CreateResult = ResultOf<CreateIndividualCommand>; // string
```

## 4. Type guards — valider l'`unknown` venant de Kafka

But : narrower une payload non fiable (message Kafka, body brut) vers un type sûr avant traitement.

```typescript
interface IndividualCreatedEvent {
  type: 'IndividualCreated';
  aggregateId: string;
  payload: IndividualEventPayload;
}

function isIndividualCreated(msg: unknown): msg is IndividualCreatedEvent {
  return (
    typeof msg === 'object' &&
    msg !== null &&
    (msg as { type?: unknown }).type === 'IndividualCreated' &&
    'aggregateId' in msg
  );
}

@EventPattern('events.individual')
handle(@Payload() raw: unknown) {
  if (!isIndividualCreated(raw)) return; // sinon on ignore proprement
  // raw est IndividualCreatedEvent ici
  this.projector.apply(raw.aggregateId, raw.payload);
}
```

## 5. Discriminated unions — domain events & dispatch event-sourcing

But : le champ discriminant `type` rend les events mutuellement exclusifs et garantit l'exhaustivité
du reducer/projector. C'est le cœur d'un typage event-sourcing sûr.

```typescript
type IndividualEvent =
  | { type: 'IndividualCreated'; aggregateId: string; payload: IndividualEventPayload }
  | { type: 'IndividualRenamed'; aggregateId: string; firstName: string; lastName: string }
  | { type: 'IndividualArchived'; aggregateId: string; reason: string };

function reduce(state: Individual, event: IndividualEvent): Individual {
  switch (event.type) {
    case 'IndividualCreated':
      return { ...event.payload, id: event.aggregateId, createdAt: new Date() };
    case 'IndividualRenamed':
      return { ...state, firstName: event.firstName, lastName: event.lastName };
    case 'IndividualArchived':
      return state;
    default:
      return assertNever(event); // compile erreur si un event est ajouté sans case
  }
}

function assertNever(x: never): never {
  throw new Error(`Event non géré: ${JSON.stringify(x)}`);
}
```

## 6. Template literal types — topics Kafka, routes, clés d'event

But : imposer le motif des chaînes au niveau du type (topics, noms d'events).

```typescript
type Aggregate = 'individual' | 'questionnaire' | 'company';
type KafkaTopic = `events.${Aggregate}`;        // 'events.individual' | ...
type EventName = `${Capitalize<Aggregate>}${'Created' | 'Updated' | 'Deleted'}`;

const topic: KafkaTopic = 'events.individual'; // 'events.individuel' → erreur ✅
```

## 7. `infer` — extraire payload et résultat

But : capturer un type interne (payload d'event, retour de handler) sans le réécrire.

```typescript
// type du payload porté par un event
type PayloadOf<E> = E extends { payload: infer P } ? P : never;
type P = PayloadOf<IndividualCreatedEvent>; // IndividualEventPayload

// type de retour résolu d'un handler asynchrone
type Awaited2<T> = T extends Promise<infer R> ? R : T;
type HandlerResult<H> = H extends { execute(...args: any[]): infer R } ? Awaited2<R> : never;
```

---

## Extras au-delà des bases

### Branded / nominal types — identifiants d'agrégat non confondables

But : empêcher de passer un `QuestionnaireId` là où un `IndividualId` est attendu, même si les deux
sont des `string`.

```typescript
type Brand<T, B> = T & { readonly __brand: B };

type IndividualId = Brand<string, 'IndividualId'>;
type QuestionnaireId = Brand<string, 'QuestionnaireId'>;

const asIndividualId = (v: string) => v as IndividualId;

function loadIndividual(id: IndividualId) { /* ... */ }
// loadIndividual(questionnaireId) → erreur de compilation ✅
```

### `satisfies` — config validée sans perdre les littéraux

But : valider la forme d'un objet de config/mapping tout en gardant les types précis.

```typescript
type LogLevel = 'debug' | 'info' | 'warn' | 'error';

const kafkaConfig = {
  clientId: 'ir-consumer',
  level: 'info',
  retries: 5
} satisfies { clientId: string; level: LogLevel; retries: number };

// kafkaConfig.level est 'info' (littéral), forme validée
```

### `const` assertions — statuts figés + union dérivée

But : geler une liste readonly et en tirer une union exploitable partout.

```typescript
const REVIEW_STATUS = ['approved', 'rejected', 'pending'] as const;

type ReviewStatus = typeof REVIEW_STATUS[number]; // 'approved' | 'rejected' | 'pending'
```

### Generics avancés — repository et handler génériques

But : une base typée réutilisable pour tous les agrégats/handlers, avec contraintes.

```typescript
interface AggregateRoot { id: string; }

abstract class BaseRepository<T extends AggregateRoot> {
  protected abstract collection: string;
  abstract findById(id: T['id']): Promise<T | null>;
  abstract save(entity: T): Promise<void>;
}

interface IHandler<TCommand, TResult> {
  execute(command: TCommand): Promise<TResult>;
}

class CreateIndividualHandler
  implements IHandler<CreateIndividualCommand, IndividualId> {
  async execute(command: CreateIndividualCommand): Promise<IndividualId> {
    // ...
    return asIndividualId('...');
  }
}
```

---

## Règles à respecter

- Pas de `any`. Tout ce qui entre par Kafka/HTTP est `unknown` jusqu'à un type guard.
- Pas de `as` sauf branding contrôlé ou cast prouvé par un guard juste avant.
- Dérive les DTOs (`Omit`/`Pick`/`Partial`) depuis l'entité de domaine, ne les duplique pas.
- Domain events en discriminated unions avec `assertNever` dans le reducer/projector.
- Branded types pour les identifiants d'agrégat.
- `satisfies` pour la config, `as const` pour les listes de statuts.
