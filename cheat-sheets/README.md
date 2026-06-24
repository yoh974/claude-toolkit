# Cheat sheets

Pense-bêtes condensés (1 page chacun) pour les technos du workspace : commandes essentielles, snippets, conventions, pièges courants. Pas de doc exhaustive — voir les `CLAUDE.md` / `.claude/rules/` de chaque repo pour le détail complet.

## backend/

| Fiche | Sujet |
|---|---|
| [nestjs-ddd-cqrs-event-sourcing.md](backend/nestjs-ddd-cqrs-event-sourcing.md) | Architecture DDD, Aggregates, Commands/Queries, Projections, Sagas |
| [prisma-mongodb.md](backend/prisma-mongodb.md) | Schémas Prisma, types composites, Event Store |
| [kafka.md](backend/kafka.md) | Structure des topics, contrats Zod, DLQ/reply |
| [zod-validation.md](backend/zod-validation.md) | Schémas de validation, inférence de type |
| [jest-testing.md](backend/jest-testing.md) | Tests unitaires (in-memory) et E2E |

## frontend/

| Fiche | Sujet |
|---|---|
| [angular-19.md](frontend/angular-19.md) | Standalone, signals, control flow, config dynamique, auth |
| [single-spa-mfe.md](frontend/single-spa-mfe.md) | Micro-frontends, Module Federation, shell/remotes |

## infra/

| Fiche | Sujet |
|---|---|
| [docker-compose-traefik-make.md](infra/docker-compose-traefik-make.md) | Routing local, profils de services, commandes make |
| [tools-cli.md](infra/tools-cli.md) | CLI de seed/ingestion/extraction Kafka |
