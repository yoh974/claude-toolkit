# Jest — Tests unitaires & E2E

## Tests unitaires

Un fichier par handler, colocalisé : `create-my.handler.spec.ts`.

Pattern AAA + "When... then..." :

```ts
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
      expect(await repository.findById(MyId.fromValue(payload.id))).toBeDefined();
    });
  });

  describe('When the entity does not exist', () => {
    it('should throw MyNotFoundException', async () => {
      await expect(handler.execute(...)).rejects.toThrow(MyNotFoundException);
    });
  });
});
```

Règles : **in-memory adapters** avec snapshots **complets** (tous champs obligatoires), `randomUUID()` de `node:crypto`, enums du domaine (pas de strings littérales), mocker tous les services injectés.

Cas obligatoires : création (minimale + complète + doublon), mise à jour (succès + erreur 404), archivage/désarchivage (succès + erreurs), sagas (rollback + idempotence).

## Tests E2E

Un fichier par endpoint HTTP / réaction Kafka, **une assertion par test** (max deux si liées).

```ts
beforeAll(async () => {
  app = await startServer({ module: { imports: [CoreModule, MyModule] } });
  repository = app.get<MyQueryRepositoryPort>(MY_QUERY_REPOSITORY);
});
afterAll(async () => { await stopServer(app); });
```

**Ne jamais mocker les repositories** — vrais repositories Prisma. Seules les API/services externes peuvent être mockés. Tests d'erreur attendus : 400, 404, 409.

## Commandes

```bash
yarn test            # unitaires
yarn test:watch
yarn test:coverage
yarn test:e2e        # utilise .env.test
npx jest path/to/file.spec.ts --no-coverage   # un seul fichier
```

## Pièges courants

- Mocker un repository dans un test E2E → masque les vraies erreurs de migration/mapping
- Snapshot in-memory incomplet (champs optionnels absents) → faux positifs
- Plusieurs assertions indépendantes dans un même test E2E → diagnostic difficile en cas d'échec
