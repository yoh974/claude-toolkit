# Zod — Validation

Validation des payloads (commandes, contrats Kafka) avec inférence de type automatique.

## Schéma défini dans le fichier de commande

```ts
export const CreateMyCommandSchema = z.object({
  id: z.string().uuid().optional(),
  requiredString: z.string(),
  optionalString: z.string().optional(),
  status: z.enum(['ACTIVE', 'INACTIVE']),
  tags: z.array(z.string()).optional(),
});

export type CreateMyCommandPayload = z.infer<typeof CreateMyCommandSchema>;
```

## Conventions

| Notation | Signification |
|---|---|
| `z.string().optional()` | champ peut être absent |
| `z.string().nullable().optional()` | champ peut être `null` ou absent |

- Toujours inférer le type avec `z.infer<typeof Schema>` — ne jamais dupliquer le type à la main
- Un schéma par commande/contrat, colocalisé avec le fichier qu'il valide

## Pièges courants

- Définir un type TS séparé du schéma Zod (drift garanti) → toujours `z.infer`
- Utiliser `.optional()` quand on veut accepter `null` côté API → utiliser `.nullable().optional()`
- Oublier de valider un payload Kafka entrant avant de construire la commande
