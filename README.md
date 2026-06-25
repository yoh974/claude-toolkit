# Claude Code Toolkit

Collection d'agents, commandes et skills réutilisables pour Claude Code, accumulés sur plusieurs projets. Contenu anonymisé : aucun nom de société, de personne, d'URL interne ou de chemin local n'apparaît — tous les exemples (org Azure DevOps, e-mails, noms de services) sont génériques.

## Structure

```
.
├── agents/             Agents génériques (dev backend NestJS, dev frontend Angular, tests, doc, git, orchestration)
├── commands/           Commandes slash génériques (structuration de prompt, brainstorm contradictoire)
├── skills/             Skills génériques réutilisables sur n'importe quel projet
├── cheat-sheets/       Pense-bêtes condensés par techno (backend/frontend/infra)
└── ddd-medical-project/  Skills/agents/règles spécifiques à un projet DDD du domaine médical (voir son README)
```

## agents/

| Agent | Rôle |
|---|---|
| `nestjs-dev-back.md` | Développeur backend NestJS senior, DDD/CQRS/Event Sourcing |
| `angular-dev-front.md` | Développeur frontend Angular, clean architecture |
| `nestjs-test.md` | Spécialiste tests unitaires NestJS/Jest |
| `doc-writer.md` | Rédacteur de documentation technique |
| `git-ops.md` | Commits atomiques + push sécurisé |
| `token-optimizer.md` | Orchestrateur de pipeline multi-agents, optimisation tokens |

## commands/

| Commande | Rôle |
|---|---|
| `ccoa.md` | Transforme un prompt brut en structure Contexte/Contraintes/Objectif/Actions, puis entre en mode plan |
| `brain-storm.md` | Avocat du diable — contredit systématiquement pour aiguiser la réflexion |

## skills/

| Skill | Rôle |
|---|---|
| `azure-devops-cli/` | Référence complète pour piloter Azure DevOps via `az` (boards, repos, pipelines) |
| `create-update-pr/` | Crée/met à jour une PR Azure DevOps avec description, work item, reviewers |
| `front-pr-review/` | Review de PR frontend Angular/Material Design |
| `review-pr/` | Review de PR contre les règles d'architecture DDD/CQRS/Event Sourcing |
| `majestic-avalanche/` | Orchestrateur de pipeline de développement automatisé (dev → test → doc → git → PR) |
| `start-task/` | Démarre une tâche Azure DevOps : crée la branche, détecte un template de pipeline automatique |
| `ts-typing-angular/` | Guide de typage TypeScript expert appliqué à Angular (signals, formulaires réactifs, models) |
| `ts-typing-nestjs/` | Guide de typage TypeScript expert appliqué à NestJS (DTOs, domain events, CQRS/Event Sourcing) |

## cheat-sheets/

Pense-bêtes 1-page par techno : backend (NestJS/DDD/CQRS/Event Sourcing, Prisma/MongoDB, Kafka, Zod, Jest), frontend (Angular 19, single-spa/Module Federation), infra (Docker Compose/Traefik/Make, CLI d'outillage). Voir [cheat-sheets/README.md](cheat-sheets/README.md).

## ddd-medical-project/

Sous-dossier dédié à un projet illustrant une architecture **DDD + CQRS + Event Sourcing appliquée à un domaine médical** (dossier individuel : identité, situation professionnelle, santé clinique/mentale, mode de vie, social). Voir [ddd-medical-project/README.md](ddd-medical-project/README.md).
