---
description: Règles pour l'organisation des schémas Prisma et les types MongoDB
globs: prisma/schema/**/*.prisma
---

# Prisma Schemas

## Organisation

```
prisma/schema/
├── base.prisma          # Types communs : FuzzyDate, Traceability, Comment, ReferentielItem
├── event.prisma          # Event Store : Event, OutboxEvent, Snapshot
├── identity/             # Schémas identité
├── work/                 # Schémas travail
├── health/               # Schémas santé
├── social/               # Schémas social
└── ...
```

## Conventions

- Un fichier ou dossier par domaine métier
- Types composites MongoDB (embedded) pour les objets imbriqués, pas de relations
- Inclure des index stratégiques pour les requêtes et le tri
- Les `@default()` Prisma ne sont PAS appliqués par le driver MongoDB natif — toujours renseigner explicitement les valeurs par défaut dans les projections

## Types de base réutilisables

- `FuzzyDate` : `{ year, month?, day? }` pour les dates imprécises
- `Traceability` : `{ author, date, sourceContext }` pour l'audit
- `Comment` : commentaires avec auteur et timestamps
- `ReferentielItem` / `FreeTextOrId` : items de référentiel ou texte libre

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

Pattern Outbox : `OutboxEvent` miroir pour la publication Kafka.
