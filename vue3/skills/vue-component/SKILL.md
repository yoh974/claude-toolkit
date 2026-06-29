---
name: vue-component
description: "Crée un composant Vue 3 SFC avec TypeScript : script setup, props typées avec defineProps, emits typés avec defineEmits, composable associé si logique non triviale."
argument-hint: "<ComponentName> [--with-composable] — ex: UserCard --with-composable"
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep"]
---

# Vue Component

Crée un composant Vue 3 SFC complet avec TypeScript strict.

**Arguments:** "$ARGUMENTS"

Parse les arguments :
- `ComponentName` — nom du composant en PascalCase (ex. `UserCard`)
- `--with-composable` — générer aussi le composable associé (optionnel)

Dériver :
- `componentKebab` — kebab-case (ex. `user-card`)
- `composableName` — `use` + ComponentName (ex. `useUserCard`)

---

## Étape 1 — Découverte

Lire les composants existants pour respecter les conventions du projet :

```bash
find src/components -name '*.vue' | head -5
```

Si des composants existants sont trouvés, en lire un pour détecter :
- La structure de dossier (flat vs dossiers par composant)
- Le pattern de props (interface inline vs fichier types.ts)
- L'utilisation de librairie UI (PrimeVue, Vuetify, shadcn-vue, etc.)

---

## Étape 2 — Créer le composant

Créer `src/components/{ComponentName}.vue` (ou `src/components/{ComponentName}/{ComponentName}.vue` si structure en dossiers) :

```vue
<script setup lang="ts">
interface Props {
  title: string
  description?: string
  isLoading?: boolean
}

interface Emits {
  (e: 'action', payload: string): void
  (e: 'close'): void
}

const props = withDefaults(defineProps<Props>(), {
  description: '',
  isLoading: false,
})

const emit = defineEmits<Emits>()

function handleAction() {
  emit('action', props.title)
}
</script>

<template>
  <div class="user-card">
    <div v-if="isLoading" class="user-card__skeleton" aria-busy="true" />
    <template v-else>
      <h2 class="user-card__title">{{ title }}</h2>
      <p v-if="description" class="user-card__description">{{ description }}</p>
      <button type="button" @click="handleAction">Action</button>
    </template>
  </div>
</template>
```

**Règles template :**
- `v-if` avant `v-for` si les deux sont sur le même élément — ou utiliser un `<template>` wrapper
- `:key` stable et unique pour les `v-for` (jamais l'index si la liste est réordonnée)
- Attributs ARIA (`aria-busy`, `aria-label`, `role`) sur les éléments interactifs
- Jamais de `v-html` sur du contenu utilisateur

---

## Étape 3 — Si `--with-composable`

Créer `src/composables/use{ComponentName}.ts` :

```typescript
import { ref, computed, onUnmounted } from 'vue'

interface UseUserCardOptions {
  initialTitle?: string
}

export function useUserCard(options: UseUserCardOptions = {}) {
  const title = ref(options.initialTitle ?? '')
  const isLoading = ref(false)
  const error = ref<Error | null>(null)

  const displayTitle = computed(() => title.value.trim() || 'Sans titre')

  async function load(id: number) {
    isLoading.value = true
    error.value = null
    try {
      const response = await fetch(`/api/items/${id}`)
      if (!response.ok) throw new Error(`HTTP ${response.status}`)
      const data = await response.json() as { title: string }
      title.value = data.title
    } catch (e) {
      error.value = e instanceof Error ? e : new Error(String(e))
    } finally {
      isLoading.value = false
    }
  }

  return {
    title,
    displayTitle,
    isLoading,
    error,
    load,
  }
}
```

---

## Étape 4 — Vérification TypeScript

```bash
npx vue-tsc --noEmit 2>&1 | head -30
```

---

## Résultat attendu

Rapporter :
- Fichiers créés
- Props et emits définis
- Composable créé si `--with-composable`
- Erreurs TypeScript détectées
