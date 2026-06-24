# DDD Medical Project — Claude Code Assets

Skills, agents et règles d'architecture pour un microservice NestJS de **dossier individuel médical** (identité, situation professionnelle, santé clinique/mentale, mode de vie, social), construit en **DDD + CQRS + Event Sourcing** sur MongoDB/Prisma, avec intégration Kafka.

Contenu anonymisé : tout repo, service ou organisation est désigné par un nom générique (`services/patient-record-service`, `your-org`, etc.).

## Structure

```
.
├── rules/      Règles d'architecture (lues par Claude Code en contexte projet)
├── agents/     Agents de revue spécialisés par couche DDD
└── skills/     Skills opérationnels (seed de données, ingestion, génération de code)
```

## rules/

Numérotées dans l'ordre de lecture recommandé : architecture générale → couches (domain/application/infra/presentation) → tests → Kafka → Zod → permissions → events/context → schémas Prisma.

| Fichier | Sujet |
|---|---|
| `00-architecture.md` | Structure des modules, dépendances entre couches, Composition Root |
| `01-standards.md` | Conventions de nommage, style de code, imports |
| `02-domain-layer.md` | Aggregates, Domain Events, Exceptions, Ports, Value Objects |
| `03-application-layer.md` | Commands, Queries, Read Models, Sagas |
| `04-infrastructure-layer.md` | Repositories (Event Sourcing + in-memory), Projections |
| `05-presentation-layer.md` | Controllers HTTP/Kafka, DTOs, Interceptors |
| `06-testing.md` | Pattern AAA, cas de test obligatoires, règles E2E |
| `07-kafka-contracts.md` | Structure des topics, contrats Zod |
| `08-zod-validation.md` | Conventions de schémas Zod pour les commandes |
| `09-authentication-permissions.md` | Décorateurs d'autorisation, permissions fines |
| `10-context-events-core.md` | Context Service (AsyncLocalStorage), types d'événements, sagas |
| `11-prisma-schemas.md` | Organisation des schémas Prisma, Event Store |

## agents/

| Agent | Rôle |
|---|---|
| `ddd-domain-reviewer.md` | Revue de la couche domaine (aggregates, events, VOs, exceptions) |
| `ddd-infra-reviewer.md` | Revue de la couche infrastructure (repositories, projections) |
| `ddd-presentation-reviewer.md` | Revue de la couche présentation (controllers, DTOs) |

## skills/

| Skill | Rôle |
|---|---|
| `add-rattachement-entity/` | Génère les artefacts pour une nouvelle entité de rattachement (VO, label, projections, index) |
| `extract-kafka-topics/` | Extrait les topics Kafka nouveaux/modifiés depuis un diff git, génère un ticket markdown |
| `review-pr-domain/` | Review de PR contre les règles d'architecture DDD/CQRS/Event Sourcing |
| `add-clinical-mental-health-data/` | Ajoute des données de santé clinique et mentale à un individu |
| `add-lifestyle-addictions-data/` | Ajoute des données de modes de vie et addictions |
| `add-social-data/` | Ajoute des données sociales (parcours, accidents, incapacités...) |
| `seed-individual-record-full/` | Seed complet d'un individu via HTTP (identité, travail, santé, social) |
| `seed-medical-visit/` | Crée une visite médicale liée à un individu |
| `seed-legacy-social/` | Insère des données legacy simulées (maladie professionnelle + reprise) |
| `seed-legacy-occupational-disease/` | Publie une maladie professionnelle legacy via Kafka |
| `import-recovery-data/` | Insère des données de reprise directement en base, en bypassant le pipeline d'ingestion |
| `run-ingestion-scope/` | Lance un scope d'ingestion sur toutes les ingestions présentes en base |
| `replay-reference-data-event/` | Rejoue des événements de création d'item référentiel vers Kafka |
