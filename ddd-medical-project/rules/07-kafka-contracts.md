---
description: Règles pour les contrats Kafka et topics
globs: src/core/kafka/**/*.ts
---

# Kafka Topics & Contracts

## Structure des dossiers :

```
src/core/kafka/kafka-topics/
└── [namespace]/[api-version]/
    ├── base.topic.ts           # Namespace + ApiVersion
    └── domain/[domain-name]/
        ├── topic.ts            # Name, DeadLetterTopic, ReplyTopic, Events
        └── events/[event].contract.ts
```

## Contract (Zod) :

```typescript
export const MyEventPayloadSchema = z.object({
  id: z.string().uuid(),
  field: z.string().max(250),
});
export type MyEventPayload = z.infer<typeof MyEventPayloadSchema>;
export const MyEventEvent = { ContentType: `${BaseTopic}.message.MyEvent` };
```

## ContentType : `${BaseTopic}.message.[EventName]`

## Règles :

- Contrats immutables une fois déployés
- Toujours valider avec le schéma Zod dans les controllers
- Breaking changes → nouvelle version d'API
