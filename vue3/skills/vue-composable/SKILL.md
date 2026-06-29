---
name: vue-composable
description: "Crée un composable Vue 3 avec TypeScript : use* pattern, gestion loading/error, cleanup onUnmounted, retour typé avec état, actions et computed."
argument-hint: "<composableName> [resource] — ex: useUserProfile User"
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep"]
---

# Vue Composable

Crée un composable Vue 3 TypeScript suivant les bonnes pratiques Composition API.

**Arguments:** "$ARGUMENTS"

Parse les arguments :
- `composableName` — nom en camelCase avec préfixe `use` (ex. `useUserProfile`)
- `resource` — entité ou ressource manipulée en PascalCase (ex. `User`) — optionnel

Dériver :
- `fileName` — `composableName.ts` (ex. `useUserProfile.ts`)
- `returnType` — interface de retour `Use{Resource}Return` (ex. `UseUserProfileReturn`)

---

## Étape 1 — Découverte

Lire les composables existants pour respecter les conventions du projet :

```bash
find src/composables -name 'use*.ts' | head -8
```

Si des composables existent, en lire un pour détecter :
- La structure de retour (objet nommé vs tableau)
- La gestion des erreurs (type Error, string, objet custom)
- L'utilisation de VueUse si disponible

Vérifier si VueUse est disponible :
```bash
grep -r "@vueuse/core" package.json 2>/dev/null
```

---

## Étape 2 — Créer le composable

Créer `src/composables/{fileName}` :

```typescript
import { ref, computed, onUnmounted } from 'vue'

interface User {
  id: number
  name: string
  email: string
}

interface UseUserProfileReturn {
  user: Readonly<Ref<User | null>>
  isLoading: Readonly<Ref<boolean>>
  error: Readonly<Ref<Error | null>>
  displayName: ComputedRef<string>
  load: (id: number) => Promise<void>
  reset: () => void
}

export function useUserProfile(): UseUserProfileReturn {
  const user = ref<User | null>(null)
  const isLoading = ref(false)
  const error = ref<Error | null>(null)

  const displayName = computed(() => user.value?.name ?? 'Utilisateur inconnu')

  let abortController: AbortController | null = null

  async function load(id: number): Promise<void> {
    abortController?.abort()
    abortController = new AbortController()

    isLoading.value = true
    error.value = null

    try {
      const response = await fetch(`/api/users/${id}`, {
        signal: abortController.signal,
      })
      if (!response.ok) throw new Error(`HTTP ${response.status}`)
      user.value = await response.json() as User
    } catch (e) {
      if (e instanceof DOMException && e.name === 'AbortError') return
      error.value = e instanceof Error ? e : new Error(String(e))
    } finally {
      isLoading.value = false
    }
  }

  function reset(): void {
    user.value = null
    isLoading.value = false
    error.value = null
  }

  onUnmounted(() => {
    abortController?.abort()
  })

  return {
    user: readonly(user),
    isLoading: readonly(isLoading),
    error: readonly(error),
    displayName,
    load,
    reset,
  }
}
```

**Règles obligatoires :**
- `readonly()` sur les refs exposées pour signaler qu'elles ne doivent pas être mutées de l'extérieur
- `AbortController` sur les `fetch` pour annuler en cas de démontage ou d'appel redondant
- `onUnmounted` (ou `onScopeDispose` si le composable peut vivre hors composant) pour le cleanup
- Retourner un **objet nommé** — jamais un tableau sauf si l'ordre est intentionnel (`[value, setter]`)

---

## Étape 3 — Si le composable est générique (factory pattern)

Pour un pattern fetch réutilisable, créer une factory :

```typescript
import { ref, readonly } from 'vue'

export function createUseFetch<T>(fetcher: (signal: AbortSignal) => Promise<T>) {
  return function useFetch() {
    const data = ref<T | null>(null)
    const isLoading = ref(false)
    const error = ref<Error | null>(null)
    let abortController: AbortController | null = null

    async function execute() {
      abortController?.abort()
      abortController = new AbortController()
      isLoading.value = true
      error.value = null
      try {
        data.value = await fetcher(abortController.signal)
      } catch (e) {
        if (e instanceof DOMException && e.name === 'AbortError') return
        error.value = e instanceof Error ? e : new Error(String(e))
      } finally {
        isLoading.value = false
      }
    }

    onUnmounted(() => abortController?.abort())

    return { data: readonly(data), isLoading: readonly(isLoading), error: readonly(error), execute }
  }
}

// Usage
export const useProducts = createUseFetch<Product[]>((signal) =>
  fetch('/api/products', { signal }).then(r => r.json())
)
```

---

## Étape 4 — Vérification TypeScript

```bash
npx vue-tsc --noEmit 2>&1 | head -30
```

---

## Résultat attendu

Rapporter :
- Fichier créé
- Signature de retour
- Effets de bord et cleanup présents
- Erreurs TypeScript détectées
