---
description: Règles d'utilisation de Zod pour la validation
globs: src/**/application/commands/**/*.command.ts
---

# Zod

## Schémas définis dans les fichiers de commande :

```typescript
export const CreateMyCommandSchema = z.object({
  id: z.string().uuid().optional(),
  requiredString: z.string(),
  optionalString: z.string().optional(),
  status: z.enum(['ACTIVE', 'INACTIVE']),
  tags: z.array(z.string()).optional(),
});

export type CreateMyCommandPayload = z.infer<typeof CreateMyCommandSchema>;
```

## Conventions :

- `z.string().optional()` : champ peut être absent
- `z.string().nullable().optional()` : champ peut être null ou absent
- Toujours inférer le type du schéma avec `z.infer`
