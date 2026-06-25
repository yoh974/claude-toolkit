---
name: ts-typing-angular
description: "Guide de typage TypeScript niveau expert pour Angular. À appliquer quand on écrit ou refactore du TypeScript dans un projet Angular (composants, signals, services, formulaires réactifs, models OpenAPI) et qu'on veut un typage robuste sans any ni cast."
---

# Typage TypeScript expert — Angular

Quand tu écris ou refactores du TypeScript dans un projet Angular, applique ces techniques.
Objectif : **zéro `any`, zéro cast (`as`) non justifié, zéro duplication de types**. L'inférence
fait le travail, le compilateur attrape les bugs.

Pour chaque technique : un ❌ (typage faible) → ✅ (typage expert). Code court, idiomatique Angular 19.

---

## 1. Utility types — dériver, ne pas dupliquer

But : transformer un type existant au lieu de le réécrire. Idéal sur les models générés OpenAPI
(`src/models/`).

```typescript
// ❌ on redéclare le payload de création à la main
interface CreateQuestionnaire {
  title: string;
  description: string;
}

// ✅ on dérive du model source — il reste synchro automatiquement
import { Questionnaire } from '@models';

type QuestionnaireDraft = Omit<Questionnaire, 'id' | 'createdAt'>;
type QuestionnairePatch = Partial<Omit<Questionnaire, 'id'>>;
type QuestionnaireListItem = Pick<Questionnaire, 'id' | 'title' | 'status'>;

// Record pour un dictionnaire fortement typé
type StatusLabels = Record<Questionnaire['status'], string>;
```

## 2. Mapped types — un FormGroup typé depuis un model

But : générer un type à partir des clés d'un autre. Évite les `FormGroup<any>` et les
`form.get('xxx')` qui renvoient `AbstractControl | null`.

```typescript
import { FormControl, FormGroup } from '@angular/forms';

// ✅ chaque clé du model devient un FormControl typé
type ControlsOf<T> = {
  [K in keyof T]: FormControl<T[K]>;
};

type QuestionnaireForm = FormGroup<ControlsOf<QuestionnaireDraft>>;

const form: QuestionnaireForm = new FormGroup({
  title: new FormControl('', { nonNullable: true }),
  description: new FormControl('', { nonNullable: true })
});

// form.controls.title est FormControl<string>, pas AbstractControl | null
const title: string = form.controls.title.value;
```

## 3. Conditional types — déballer Signal / Observable

But : faire dépendre un type d'une condition. Utile pour extraire la valeur portée par un wrapper.

```typescript
import { Signal } from '@angular/core';
import { Observable } from 'rxjs';

type Unwrap<T> =
  T extends Signal<infer V> ? V :
  T extends Observable<infer V> ? V :
  T;

type A = Unwrap<Signal<number>>;      // number
type B = Unwrap<Observable<User[]>>;  // User[]
type C = Unwrap<string>;              // string
```

## 4. Type guards — narrower les réponses d'API

But : affiner `unknown` / une union vers un type précis dans un bloc. Indispensable côté front où
les données entrantes ne sont pas fiables.

```typescript
interface ApiError { error: string; code: number; }

function isApiError(value: unknown): value is ApiError {
  return (
    typeof value === 'object' &&
    value !== null &&
    'error' in value &&
    'code' in value
  );
}

// usage dans un catchError / un effect
if (isApiError(payload)) {
  // payload est ApiError ici, autocomplétion complète
  this.toast.error(payload.error);
}
```

## 5. Discriminated unions — modéliser l'état de vue

But : un champ discriminant (`status`) rend les états mutuellement exclusifs. Le template ne peut
plus lire `data` quand on charge encore. Couplé à `computed()`, c'est le pattern propre pour les
vues async.

```typescript
type ViewState<T> =
  | { status: 'loading' }
  | { status: 'loaded'; data: T }
  | { status: 'error'; message: string };

readonly state = signal<ViewState<Questionnaire[]>>({ status: 'loading' });

// dans le template, @switch (state().status) garantit l'exhaustivité
readonly itemCount = computed(() => {
  const s = this.state();
  return s.status === 'loaded' ? s.data.length : 0; // s.data inaccessible sinon
});
```

Astuce exhaustivité — le compilateur refuse un cas oublié :

```typescript
function assertNever(x: never): never {
  throw new Error(`État non géré: ${JSON.stringify(x)}`);
}
```

## 6. Template literal types — events, routes, clés i18n typés

But : imposer un motif de chaîne au niveau du type.

```typescript
// noms d'events custom
type EntityEvent = `questionnaire:${'created' | 'updated' | 'deleted'}`;

// routes typées
type AppRoute = `/qt-answer/${string}` | '/dashboard' | `/questionnaire/${string}/edit`;

// clés de traduction dérivées
type TranslationKey = `questionnaire.${'title' | 'description' | 'status'}`;
```

## 7. `infer` — extraire un type depuis une structure

But : capturer un type interne sans le réécrire.

```typescript
// type émis par un computed/signal sans le déclarer en double
type SignalValue<T> = T extends Signal<infer V> ? V : never;

// type des arguments d'une méthode de service (pour un wrapper générique)
type FirstArg<F> = F extends (arg: infer A, ...rest: any[]) => any ? A : never;
```

---

## Extras au-delà des bases

### Branded / nominal types — des IDs non interchangeables

But : empêcher de passer un `QuestionnaireId` là où un `UserId` est attendu, même si les deux sont
`string`.

```typescript
type Brand<T, B> = T & { readonly __brand: B };

type QuestionnaireId = Brand<string, 'QuestionnaireId'>;
type UserId = Brand<string, 'UserId'>;

const asQuestionnaireId = (v: string) => v as QuestionnaireId;

function load(id: QuestionnaireId) { /* ... */ }
load(asQuestionnaireId(route.params['id']));
// load(userId) → erreur de compilation ✅
```

### `satisfies` — valider sans perdre l'inférence littérale

But : vérifier qu'un objet respecte une forme tout en gardant les types littéraux précis. Parfait
pour la config runtime (`app-config.json`, `environment`).

```typescript
type AuthMode = 'no' | 'custom' | 'mfe';

const config = {
  production: false,
  auth: 'mfe',
  apiUrl: 'https://api.example.com'
} satisfies { production: boolean; auth: AuthMode; apiUrl: string };

// config.auth est 'mfe' (littéral), pas string — et la forme est validée
```

❌ `const config: Config = {...}` élargirait `auth` à `AuthMode` et perdrait le littéral.

### `const` assertions — listes figées + union dérivée

But : geler une valeur en readonly et en tirer une union de types.

```typescript
const QUESTION_TYPES = ['text', 'choice', 'date', 'number'] as const;

type QuestionType = typeof QUESTION_TYPES[number]; // 'text' | 'choice' | 'date' | 'number'

// itérable dans un @for, et typé pour un mat-select
```

### Generics avancés — service/composant réutilisable

But : une seule implémentation typée pour N entités, avec contraintes.

```typescript
interface Entity { id: string; }

@Injectable()
export class CrudStore<T extends Entity> {
  private readonly items = signal<T[]>([]);

  readonly all = this.items.asReadonly();

  upsert(item: T): void {
    this.items.update(list => {
      const i = list.findIndex(x => x.id === item.id);
      return i === -1 ? [...list, item] : list.map(x => (x.id === item.id ? item : x));
    });
  }

  byId(id: T['id']): T | undefined {
    return this.items().find(x => x.id === id);
  }
}
```

---

## Règles à respecter

- Pas de `any`. Utilise `unknown` + type guard si l'entrée est incertaine.
- Pas de `as` sauf branding contrôlé ou cast vers un type plus précis prouvé par un guard juste avant.
- Dérive (`Omit`/`Pick`/`Partial`) plutôt que dupliquer les models OpenAPI.
- Préfère `satisfies` à l'annotation explicite pour les objets de config.
- `FormControl` typés (`nonNullable: true`) plutôt que `FormGroup<any>`.
- Discriminated unions pour tout état multi-cas (chargement, résultat de règle, étape de wizard).
