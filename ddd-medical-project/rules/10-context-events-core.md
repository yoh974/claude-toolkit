---
description: Règles pour le Context Service, les types d'événements et les patterns core
globs: src/core/**/*.ts
---

# Core Patterns

## Context Service (AsyncLocalStorage)

Contexte request-scoped propagé via `AsyncLocalStorage`, transport-agnostique :

```typescript
type Context<TMetadata> = {
  requestId?: string;
  source: 'http' | 'kafka' | 'grpc' | 'job';
  user?: AuthenticatedUser;
  token?: string;
  tenantId?: string;
  saga?: SagaContext;
  withConsistency?: boolean;
  metadata?: TMetadata;
};
```

- HTTP : initialisé par `HTTPRequestContextMiddleware`
- Kafka : initialisé par `KafkaMessageContextInterceptor`
- Accès : `ContextService.getContext()`, `tryGetContext()`, `setContextProperty()`

## Types d'Événements

Trois types distincts :

| Type | Usage | Champs spécifiques |
|------|-------|-------------------|
| `Event<T>` | Base commune | `id, content_type, created_at, created_by, metadata, payload` |
| `DomainEvent<T>` | Event Sourcing | + `aggregate_id, aggregate_type, version` |
| `MessageEvent<T>` | Kafka request-response | + `reply_to` |
| `ErrorEvent<T>` | Réponses d'erreur | + `erred_event_id, error_type, message` |

## Event Store Service

- `saveEvents()` retourne un callback de transaction (exécution différée)
- Pattern Outbox : events sauvés dans `events` ET `outbox_events`
- Gestion des conflits de concurrence : `P2002` (unicité), `P2034` (write conflict)
- Conversion automatique Date MongoDB → ISO string

## Aggregate Root (Base)

- `apply(event)` incrémente la version automatiquement
- `buildEvent()` enrichit les événements avec UUID, timestamps, aggregate info, metadata
- Prototype setting pour les events chargés (dispatch des méthodes `on*`)

## Saga Context

```typescript
{ 'correlation-id': string, 'started-at': string }
```

Propagé dans les metadata Kafka pour le suivi et la cohérence au replay.

## MongoDB Bulk Writer

Pour les opérations haut débit (imports batch) :
- Utilise le driver natif MongoDB (bypass Prisma)
- `insertMany()` avec `ordered: false` pour la parallélisation
- Continue malgré les erreurs de clé dupliquée

## Logger (Winston)

- `WinstonLoggerFactory(appName, contextService)` crée un logger context-aware
- Inclut automatiquement : requestId, userId, tenantId, duration
- JSON en production, couleurs en développement
