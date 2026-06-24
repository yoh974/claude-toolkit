# Kafka — Topics & Contrats

Communication inter-services par événements Kafka, contrats versionnés et validés par Zod.

## Structure des dossiers

```
src/core/kafka/kafka-topics/
└── [namespace]/[api-version]/
    ├── base.topic.ts            # Namespace + ApiVersion
    └── domain/[domain-name]/
        ├── topic.ts             # Name, DeadLetterTopic, ReplyTopic, Events
        └── events/[event].contract.ts
```

## Contrat (Zod)

```ts
export const MyEventPayloadSchema = z.object({
  id: z.string().uuid(),
  field: z.string().max(250),
});
export type MyEventPayload = z.infer<typeof MyEventPayloadSchema>;
export const MyEventEvent = { ContentType: `${BaseTopic}.message.MyEvent` };
```

`ContentType` = `${BaseTopic}.message.[EventName]`.

## Domain topics vs Command/event topics

- **Domain topic** (`topic.ts`) : chaque `Name:` génère **2** topics → le topic lui-même + son DLT (`.dlt`)
- **Command/event topic** (`*.contract.ts`) : **1** topic, pas de DLT. Exclure les `ReplyTopic` (pas un topic à transmettre)

## RPC Controller (consommateur)

```ts
@MessagePattern(...)
@EventContentTypePattern(MyEventEvent.ContentType)
handle(@Payload() payload: unknown) {
  const parsed = MyEventPayloadSchema.parse(payload); // validation Zod obligatoire avant la commande
  return this.commandBus.execute(new MyCommand(parsed));
}
```
Error handling : `'dead-letter-queue'` ou `'reply'`. Aucune logique métier dans le controller.

## Pièges courants

- Modifier un contrat **déjà déployé** → interdit, créer une nouvelle version d'API à la place
- Inclure un `ReplyTopic` dans la liste des topics à transmettre aux devops → à exclure
- Oublier le topic `.dlt` associé à un nouveau domain topic
- Sauter la validation Zod dans un controller Kafka avant d'exécuter la commande
