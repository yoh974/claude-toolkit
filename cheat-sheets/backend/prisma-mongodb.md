# Prisma + MongoDB

Prisma comme ORM sur MongoDB (replica set), utilisé en event store + read models.

## Organisation des schémas

```
prisma/schema/
├── base.prisma     # FuzzyDate, Traceability, Comment, ReferentielItem
├── event.prisma     # Event, OutboxEvent, Snapshot
├── identity/ work/ health/ social/ ...
```

Un fichier/dossier par domaine métier. Types composites MongoDB (embedded) pour les objets imbriqués — **pas de relations**.

## Types de base réutilisables

| Type | Usage |
|---|---|
| `FuzzyDate` | `{ year, month?, day? }` — date imprécise |
| `Traceability` | `{ author, date, sourceContext }` — audit |
| `Comment` | commentaire + auteur + timestamps |
| `ReferentielItem` / `FreeTextOrId` | item de référentiel ou texte libre |

## Event Store

```prisma
model Event {
  id             String @id @default(auto()) @map("_id") @db.ObjectId
  aggregate_id   String
  aggregate_type String
  version        Int
  content_type   String
  payload        Json
  metadata       Json
  created_at     DateTime
  created_by     String
  @@unique([aggregate_id, version])
}
```

Pattern Outbox : `OutboxEvent` miroir, statuts `PENDING` / `PROCESSING` / `FAILED`, publié vers Kafka.

## Commandes

```bash
yarn prisma:generate   # génère le client
yarn prisma:push       # sync schema → MongoDB
yarn prisma:studio     # GUI
```

## Conversions courantes

```ts
createdAt: record.createdAt.toISOString(),   // Prisma Date → string
createdAt: new Date(payload.createdAt),      // string → Prisma Date
optionalField: record.optionalField ?? undefined, // null Prisma → undefined
```

## Pièges courants

- **`@default()` n'est PAS appliqué** par le driver MongoDB natif → toujours renseigner explicitement les valeurs par défaut dans les `build*` des projections, sinon champ manquant en base
- Modéliser une relation Mongo comme en SQL → utiliser un type embedded, pas une "relation" Prisma classique
- Oublier `@@unique([aggregate_id, version])` sur l'event store → conflits de concurrence (`P2002`) non détectés
