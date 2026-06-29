# Symfony API Platform Toolkit

Agents et skills prêts à l'emploi pour des projets **API Platform 3.x** sur Symfony 7.x.
Couvre : Resources, State Providers/Processors, Filters, OpenAPI, groupes de sérialisation.

Copiez les agents dans `.claude/agents/` et les hooks dans `.claude/settings.json` de votre projet.
À utiliser en complément du dossier `symfony/` pour les agents et skills de base Symfony.

---

## Agent

| Agent | Rôle | Modèle |
|---|---|---|
| `api-platform-dev` | Expert API Platform 3.x — Resources, Providers, Processors, Filters, OpenAPI | sonnet |
| `api-platform-qa` | Ingénieur QA ISTQB — tests d'intégration (Providers/Processors), tests HTTP JSON-LD, sérialisation, filtres, sécurité par opération | sonnet |

---

## Skills

| Skill | Usage |
|---|---|
| `/apip-resource Post` | Ajoute `#[ApiResource]` sur une Entity avec toutes les opérations, groupes de sérialisation |
| `/apip-provider Post both` | Crée ItemProvider et/ou CollectionProvider typés |
| `/apip-processor Post persist` | Crée un State Processor (persist, delete, ou async Messenger) |
| `/apip-filter Post search` | Configure des filtres built-in (`SearchFilter`, `OrderFilter`, `DateFilter`) ou un filtre custom |

---

## Hook OpenAPI

Le fichier `hooks/settings.example.json` contient un hook `PostToolUse` qui rappelle de regénérer la documentation OpenAPI après chaque modification d'un fichier PHP contenant `#[ApiResource]`.

```bash
php bin/console api:openapi:export --output=public/api.json
```

Fusionner avec `.claude/settings.json` de votre projet.

> Prérequis : `jq` installé, API Platform 3.x installé (`composer require api`).
