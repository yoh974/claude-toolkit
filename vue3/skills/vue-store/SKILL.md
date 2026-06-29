---
name: vue-store
description: "Crée un Pinia store Vue 3 avec TypeScript : setup store pattern, state typé, actions async avec gestion d'erreur, getters computed, reset."
argument-hint: "<StoreName> [EntityType] — ex: useCartStore CartItem"
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep"]
---

# Vue Store (Pinia)

Crée un Pinia store Vue 3 TypeScript suivant le setup store pattern.

**Arguments:** "$ARGUMENTS"

Parse les arguments :
- `StoreName` — nom du store en camelCase avec préfixe `use` et suffixe `Store` (ex. `useCartStore`)
- `EntityType` — type principal manipulé en PascalCase (ex. `CartItem`) — optionnel

Dériver :
- `storeId` — sans `use` et `Store`, en camelCase (ex. `cart`)
- `fileName` — `{StoreName}.ts` (ex. `useCartStore.ts`)

---

## Étape 1 — Découverte

Lire les stores existants pour respecter les conventions :

```bash
find src/stores src/**/store -name 'use*Store.ts' 2>/dev/null | head -8
```

Vérifier la version de Pinia :
```bash
grep '"pinia"' package.json
```

Vérifier si des types partagés existent :
```bash
find src/types -name '*.ts' 2>/dev/null | head -5
```

---

## Étape 2 — Créer le type (si non existant)

Créer ou compléter `src/types/{EntityType}.ts` si le type n'existe pas :

```typescript
export interface CartItem {
  id: number
  productId: number
  name: string
  quantity: number
  unitPrice: number
}

export interface Cart {
  items: CartItem[]
  couponCode: string | null
}
```

---

## Étape 3 — Créer le store

Créer `src/stores/{fileName}` (ou `src/features/{domain}/store/{fileName}`) :

```typescript
import { ref, computed } from 'vue'
import { defineStore } from 'pinia'
import type { CartItem } from '@/types/CartItem'

export const useCartStore = defineStore('cart', () => {
  // --- State ---
  const items = ref<CartItem[]>([])
  const isLoading = ref(false)
  const error = ref<Error | null>(null)
  const couponCode = ref<string | null>(null)

  // --- Getters ---
  const itemCount = computed(() => items.value.reduce((sum, i) => sum + i.quantity, 0))

  const totalPrice = computed(() =>
    items.value.reduce((sum, i) => sum + i.unitPrice * i.quantity, 0)
  )

  const isEmpty = computed(() => items.value.length === 0)

  // --- Actions ---
  async function loadCart(): Promise<void> {
    isLoading.value = true
    error.value = null
    try {
      const response = await fetch('/api/cart')
      if (!response.ok) throw new Error(`HTTP ${response.status}`)
      items.value = await response.json() as CartItem[]
    } catch (e) {
      error.value = e instanceof Error ? e : new Error(String(e))
    } finally {
      isLoading.value = false
    }
  }

  async function addItem(productId: number, quantity: number): Promise<void> {
    isLoading.value = true
    error.value = null
    try {
      const response = await fetch('/api/cart/items', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ productId, quantity }),
      })
      if (!response.ok) throw new Error(`HTTP ${response.status}`)
      const newItem = await response.json() as CartItem
      items.value = [...items.value, newItem]
    } catch (e) {
      error.value = e instanceof Error ? e : new Error(String(e))
    } finally {
      isLoading.value = false
    }
  }

  function removeItem(itemId: number): void {
    items.value = items.value.filter(i => i.id !== itemId)
  }

  function $reset(): void {
    items.value = []
    isLoading.value = false
    error.value = null
    couponCode.value = null
  }

  return {
    // State (readonly depuis l'extérieur via convention)
    items,
    isLoading,
    error,
    couponCode,
    // Getters
    itemCount,
    totalPrice,
    isEmpty,
    // Actions
    loadCart,
    addItem,
    removeItem,
    $reset,
  }
})
```

**Règles obligatoires :**
- Setup store (`() => {}`) plutôt qu'Options store pour TypeScript optimal
- `$reset()` toujours présent pour faciliter les tests
- Actions asynchrones : toujours `try/catch/finally` avec `isLoading` et `error`
- Jamais muter le state d'un autre store directement — passer par ses actions
- `storeId` unique dans toute l'application

---

## Étape 4 — Exemple d'utilisation dans un composant

```vue
<script setup lang="ts">
import { onMounted } from 'vue'
import { useCartStore } from '@/stores/useCartStore'

const cart = useCartStore()

onMounted(() => cart.loadCart())
</script>

<template>
  <div>
    <p v-if="cart.isLoading">Chargement...</p>
    <p v-else-if="cart.error">{{ cart.error.message }}</p>
    <p v-else-if="cart.isEmpty">Panier vide</p>
    <ul v-else>
      <li v-for="item in cart.items" :key="item.id">
        {{ item.name }} × {{ item.quantity }}
      </li>
    </ul>
    <p>Total : {{ cart.totalPrice }} €</p>
  </div>
</template>
```

---

## Étape 5 — Vérification TypeScript

```bash
npx vue-tsc --noEmit 2>&1 | head -30
```

---

## Résultat attendu

Rapporter :
- Fichiers créés (store + types si nouveau)
- State, getters et actions définis
- Erreurs TypeScript détectées
