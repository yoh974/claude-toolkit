# Symfony Toolkit

Agents et skills prêts à l'emploi pour des projets **PHP 8.3+ / Symfony 7.x** suivant les bonnes pratiques (PSR-12, SOLID, DDD, Doctrine ORM, PHPUnit).

Copiez les agents dans `.claude/agents/` et les hooks dans `.claude/settings.json` de votre projet.

---

## Agents

| Agent | Rôle | Modèle |
|---|---|---|
| `symfony-dev` | Développeur PHP/Symfony senior — implémente features, services, repositories, entities | sonnet |
| `symfony-architect` | Expert architecture hexagonale, DDD, Messenger, sécurité — ADR et conseils | sonnet |
| `symfony-reviewer` | Relecteur automatique déclenché au `git commit` — analyse le diff stagé | sonnet |
| `symfony-doc` | Rédacteur PHPDoc, ADR et README pour bundles | haiku |

---

## Skills

| Skill | Usage |
|---|---|
| `/symfony-voter VoterName EntityClass` | Crée un Voter Symfony complet avec constantes, `supports()`, `voteOnAttribute()` |
| `/symfony-controller ControllerName Entity` | Crée Controller + DTOs avec `#[MapRequestPayload]`, `#[IsGranted]`, REST complet |
| `/symfony-scheduler MessageClass '0 8 * * *'` | Crée Message + Schedule + Handler + config Messenger |
| `/symfony-workflow workflow-name EntityClass` | Configure un Workflow ou StateMachine avec YAML, guards, listeners |
| `/symfony-webhook stripe` | Implémente un récepteur webhook (RequestParser, RemoteEvent, Consumer) |

---

## Hook de review automatique

Le fichier `hooks/settings.example.json` contient un hook `PreToolUse` qui injecte le diff stagé dans le contexte de Claude juste avant un `git commit`, demandant au modèle de d'abord invoquer l'agent `symfony-reviewer`.

Fusionner avec `.claude/settings.json` ou `.claude/settings.local.json` de votre projet.

> Prérequis : `jq` installé, agent `symfony-reviewer` présent dans `.claude/agents/`.
