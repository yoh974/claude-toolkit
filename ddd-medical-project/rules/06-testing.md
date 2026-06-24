---
description: Règles pour les tests unitaires et E2E (Jest)
globs: '{src/**/*.spec.ts,test/**/*.e2e-spec.ts}'
---

# Testing

## Tests Unitaires

### Organisation :

Un fichier de test par handler : `create-my.handler.spec.ts` dans le même dossier.

### Pattern AAA (Arrange, Act, Assert) + "When... then..." :

```typescript
describe('CreateMyHandler', () => {
  let handler: CreateMyHandler;
  let repository: InMemoryMyCommandRepositoryAdapter;

  beforeEach(() => {
    repository = new InMemoryMyCommandRepositoryAdapter();
    handler = new CreateMyHandler(repository);
  });

  describe('When creating with valid data', () => {
    it('should create the entity successfully', async () => {
      await handler.execute(new CreateMyCommand(payload));
      const entity = await repository.findById(MyId.fromValue(payload.id));
      expect(entity).toBeDefined();
    });
  });

  describe('When the entity does not exist', () => {
    it('should throw MyNotFoundException', async () => {
      await expect(handler.execute(...)).rejects.toThrow(MyNotFoundException);
    });
  });
});
```

### Règles critiques :

- Utiliser des **in-memory adapters** avec des snapshots **COMPLETS** (tous les champs obligatoires)
- Utiliser `randomUUID()` de `node:crypto` pour les identifiants
- Utiliser les enums du domaine, pas des strings littérales
- Mocker tous les services injectés

### Cas de test obligatoires :

- **Création** : données minimales, données complètes, erreur si existe déjà
- **Mise à jour** : succès + vérification valeurs, erreur si non trouvé
- **Archivage** : succès + vérification `isArchived`, erreur si déjà archivé, erreur si non trouvé
- **Désarchivage** : succès + `isArchived: false`, erreur si non archivé, erreur si non trouvé
- **Sagas** : tous sous-agrégats créés, rollback si erreur, idempotence

## Tests E2E

### Principe : une assertion par test (max deux si liées)

### Organisation : un fichier par endpoint HTTP / réaction Kafka

```
test/
└── my-module/
    ├── create-my.e2e-spec.ts      # POST /v1/my-resource
    ├── update-my.e2e-spec.ts      # PUT /v1/my-resource/:id
    └── get-my.e2e-spec.ts         # GET /v1/my-resource
```

### Setup :

```typescript
beforeAll(async () => {
  serverOptions = { module: { imports: [CoreModule, MyModule] } };
  app = await startServer(serverOptions);
  repository = app.get<MyQueryRepositoryPort>(MY_QUERY_REPOSITORY);
});
afterAll(async () => {
  await stopServer(app, serverOptions);
});
```

### Règles critiques :

- **NE JAMAIS mocker les repositories** avec des in-memory dans les E2E
- Utiliser les vrais repositories Prisma
- Seules les APIs/services EXTERNES peuvent être mockés
- Kafka : `kafkaClient.emit()` pour DLQ handlers, `kafkaClient.send()` pour reply handlers

### Tests de cas d'erreur : 400, 404, 409
