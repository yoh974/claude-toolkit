# Angular 19

Micro-frontends Angular 19 standalone, embarqués via single-spa (voir [single-spa-mfe.md](single-spa-mfe.md)).

## Standalone only

Pas de `NgModule`. Tous les composants `standalone: true` (implicite en Angular 19), `ChangeDetectionStrategy.OnPush` par défaut pour les composants présentationnels.

## Signals

```ts
// Inputs/outputs (remplacent @Input/@Output)
value = input<string>();
required = input.required<string>();
changed = output<string>();

// State
private readonly _items = signal<Item[]>([]);
readonly items = this._items.asReadonly();
readonly count = computed(() => this._items().length);
```

Pattern store : signal privé + `asReadonly()` exposé + `computed()` pour les sélecteurs dérivés.

## Control flow (jamais `*ngIf`/`*ngFor`)

```html
@if (loading()) { <spinner/> } @else { <content/> }
@for (item of items(); track item.id) { <row [item]="item"/> }
@switch (status()) { @case ('done') { ... } @default { ... } }
```

## Injection

`inject()` dans les services/composants plutôt que constructeur :
```ts
private readonly http = inject(HttpClient);
```

## Configuration dynamique (3 couches)

1. Build-time : `src/environments/environment.ts`
2. Runtime : `assets/app-config.json` chargé via `APP_INITIALIZER` dans `app.config.ts`
3. Post-déploiement : override `window.__env_<app>` via `env.js`

## Auth

Trois modes configurés en runtime : `no` (dev local), `custom` (popup basic auth), `mfe` (token `localStorage` injecté par le shell OIDC). Géré par un `authInterceptor` unique qui couvre les 3 modes.

## Commandes

```bash
npm start                  # dev standalone
npm run start:single-spa   # mode MFE
npm run build              # build production MFE
npm run generate:api       # régénère les types TS depuis l'OpenAPI spec
```

## Style

Prettier 120 colonnes, 2 espaces, pas de virgule finale.

## Pièges courants

- Souscription manuelle sans `takeUntilDestroyed()` → fuite mémoire
- Logique complexe ou appel de fonction dans le template → préférer `computed()`
- Oublier `track` sur un `@for` → re-render complet de la liste
- Créer un `NgModule` "pour faire simple" → toujours standalone
