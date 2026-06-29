---
name: vue-page
description: "Crée une page Vue 3 avec Vue Router 4 : vue SFC typée, définition de route en lazy loading, meta typée, guard de navigation si besoin, layout associé."
argument-hint: "<PageName> [routePath] — ex: UserProfilePage /users/:id"
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep"]
---

# Vue Page

Crée une page Vue 3 avec sa configuration Vue Router 4.

**Arguments:** "$ARGUMENTS"

Parse les arguments :
- `PageName` — nom de la page en PascalCase avec suffixe `Page` ou `View` (ex. `UserProfilePage`)
- `routePath` — chemin de la route (ex. `/users/:id`) — optionnel, déduit du nom si absent

Dériver :
- `routeName` — kebab-case sans `Page`/`View` (ex. `user-profile`)
- `fileName` — `{PageName}.vue`

---

## Étape 1 — Découverte

Lire la structure du router existant :

```bash
find src/router -name '*.ts' | head -8
cat src/router/index.ts 2>/dev/null || cat src/router/routes.ts 2>/dev/null
```

Lire les pages existantes pour le pattern :

```bash
find src/views src/**/views -name '*.vue' 2>/dev/null | head -5
```

Vérifier les layouts disponibles :

```bash
find src/layouts -name '*.vue' 2>/dev/null | head -5
```

---

## Étape 2 — Créer la page

Créer `src/views/{PageName}.vue` (ou dans le dossier feature si applicable) :

```vue
<script setup lang="ts">
import { onMounted } from 'vue'
import { useRoute } from 'vue-router'
import { useUserProfile } from '@/composables/useUserProfile'

const route = useRoute()

// Récupérer le paramètre de route avec type
const userId = computed(() => Number(route.params.id))

const { user, isLoading, error, load } = useUserProfile()

onMounted(() => load(userId.value))
</script>

<template>
  <main class="user-profile-page">
    <div v-if="isLoading" class="page-loading" aria-busy="true">
      Chargement...
    </div>

    <div v-else-if="error" class="page-error" role="alert">
      {{ error.message }}
    </div>

    <template v-else-if="user">
      <h1>{{ user.name }}</h1>
      <p>{{ user.email }}</p>
    </template>
  </main>
</template>
```

---

## Étape 3 — Ajouter la route

Localiser le fichier de routes et ajouter l'entrée en lazy loading :

```typescript
// Dans src/router/routes.ts ou src/router/index.ts

import type { RouteRecordRaw } from 'vue-router'

const routes: RouteRecordRaw[] = [
  // ... routes existantes
  {
    path: '/users/:id',
    name: 'user-profile',
    component: () => import('@/views/UserProfilePage.vue'),
    meta: {
      requiresAuth: true,
      layout: 'default',
      title: 'Profil utilisateur',
    },
    props: true,  // passe les params de route comme props si le composant les accepte
  },
]
```

**Règles obligatoires :**
- **Jamais** d'import statique pour une page — toujours `() => import(...)`
- Utiliser `name` pour la navigation — jamais de hard-coding de path dans les composants
- `meta` avec `requiresAuth` si la page est protégée
- `props: true` pour les routes avec params si les props sont déclarées dans la page

---

## Étape 4 — Si la page est protégée (meta.requiresAuth)

Vérifier si un guard d'authentification existe :

```bash
find src/router/guards -name '*.ts' 2>/dev/null
```

Si le guard n'existe pas, créer `src/router/guards/authGuard.ts` :

```typescript
import type { NavigationGuardNext, RouteLocationNormalized } from 'vue-router'
import { useAuthStore } from '@/stores/useAuthStore'

export function authGuard(
  to: RouteLocationNormalized,
  _from: RouteLocationNormalized,
  next: NavigationGuardNext,
): void {
  if (!to.meta.requiresAuth) {
    next()
    return
  }

  const auth = useAuthStore()
  if (auth.isAuthenticated) {
    next()
  } else {
    next({ name: 'login', query: { redirect: to.fullPath } })
  }
}
```

Et dans `src/router/index.ts` :
```typescript
import { authGuard } from './guards/authGuard'
router.beforeEach(authGuard)
```

---

## Étape 5 — Typer les RouteMeta (si meta personnalisée)

Si des champs de `meta` sont ajoutés, augmenter le module :

```typescript
// src/types/router.d.ts
import 'vue-router'

declare module 'vue-router' {
  interface RouteMeta {
    requiresAuth?: boolean
    layout?: 'default' | 'auth' | 'empty'
    title?: string
  }
}
```

---

## Étape 6 — Vérification

```bash
npx vue-tsc --noEmit 2>&1 | head -20
```

---

## Résultat attendu

Rapporter :
- Fichier de page créé
- Route ajoutée (nom, chemin, lazy loading, meta)
- Guard créé ou détecté
- Augmentation de RouteMeta si nécessaire
