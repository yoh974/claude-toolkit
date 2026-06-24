# tools/ — CLI de développement

CLI TypeScript (pnpm + tsx) pour les tâches de dev répétitives : seed de données, traitement d'ingestion, extraction de topics Kafka.

## Invocation

```bash
npx tsx tools/index.ts <command> [options]
```

## Commandes

| Commande | Rôle |
|---|---|
| `ir --total <n>` | Seed de données de test (génère `n` enregistrements via HTTP) |
| `ig <ingestion-id>` | Traite un scope d'ingestion donné |
| `pull-topics <cluster>` | Récupère la liste des topics Kafka via l'API Kafka UI |

```bash
npx tsx tools/index.ts ir --total 10
npx tsx tools/index.ts ig <ingestion-id>
npx tsx tools/index.ts pull-topics <cluster>
```

## Conventions

- Validation des arguments via Zod
- Toutes les opérations de seed passent par HTTP (pas d'accès direct à MongoDB/Kafka), sauf commandes explicitement marquées "legacy"/"recovery"

## Pièges courants

- Lancer un seed massif (`--total` élevé) sur un environnement partagé sans vérifier d'abord la cible (host de l'API)
- Oublier que `pull-topics` dépend de l'API Kafka UI exposée — vérifier qu'elle tourne avant d'appeler la commande
