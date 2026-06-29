---
name: vue-form
description: "Crée un formulaire Vue 3 avec VeeValidate + Zod : schéma Zod typé, useForm, Field/ErrorMessage, submit handler async, gestion des erreurs serveur."
argument-hint: "<FormName> [fields...] — ex: LoginForm email password"
allowed-tools: ["Read", "Write", "Edit", "Bash", "Glob", "Grep"]
---

# Vue Form

Crée un formulaire Vue 3 avec VeeValidate 4 + Zod.

**Arguments:** "$ARGUMENTS"

Parse les arguments :
- `FormName` — nom du formulaire en PascalCase avec suffixe `Form` (ex. `LoginForm`)
- `fields` — noms des champs en camelCase séparés par des espaces (ex. `email password`) — optionnel

Dériver :
- `fileName` — `{FormName}.vue`
- `schemaName` — `{FormName}Schema` (ex. `LoginFormSchema`)

---

## Étape 1 — Découverte

Vérifier les dépendances disponibles :

```bash
grep -E '"vee-validate"|"zod"|"@vee-validate/zod"' package.json
```

Lire les formulaires existants pour le pattern :

```bash
find src -name '*Form*.vue' 2>/dev/null | head -5
```

Si VeeValidate ou Zod n'est pas installé, signaler à l'utilisateur :
```
npm install vee-validate @vee-validate/zod zod
```

---

## Étape 2 — Créer le schéma Zod

Créer ou compléter `src/schemas/{FormName}Schema.ts` :

```typescript
import { z } from 'zod'

export const LoginFormSchema = z.object({
  email: z
    .string()
    .min(1, 'L\'email est requis')
    .email('Format email invalide'),
  password: z
    .string()
    .min(8, 'Le mot de passe doit contenir au moins 8 caractères')
    .max(128, 'Le mot de passe est trop long'),
})

export type LoginFormValues = z.infer<typeof LoginFormSchema>
```

**Règles Zod :**
- Messages d'erreur en français par défaut dans le schéma
- `z.infer<typeof Schema>` pour dériver le type — jamais définir le type manuellement en doublon
- Schéma dans un fichier séparé pour être réutilisable (API + formulaire)

---

## Étape 3 — Créer le composant de formulaire

Créer `src/components/{FormName}.vue` (ou dans le dossier feature) :

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { useForm } from 'vee-validate'
import { toTypedSchema } from '@vee-validate/zod'
import { LoginFormSchema, type LoginFormValues } from '@/schemas/LoginFormSchema'

interface Emits {
  (e: 'success', values: LoginFormValues): void
  (e: 'cancel'): void
}

const emit = defineEmits<Emits>()

const serverError = ref<string | null>(null)
const isSubmitting = ref(false)

const { handleSubmit, defineField, errors, resetForm } = useForm({
  validationSchema: toTypedSchema(LoginFormSchema),
})

const [email, emailAttrs] = defineField('email')
const [password, passwordAttrs] = defineField('password')

const onSubmit = handleSubmit(async (values) => {
  serverError.value = null
  isSubmitting.value = true

  try {
    // Appel API — à remplacer par l'action Pinia ou le composable approprié
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(values),
    })

    if (!response.ok) {
      const data = await response.json() as { message?: string }
      serverError.value = data.message ?? 'Une erreur est survenue'
      return
    }

    emit('success', values)
    resetForm()
  } catch {
    serverError.value = 'Impossible de contacter le serveur'
  } finally {
    isSubmitting.value = false
  }
})
</script>

<template>
  <form novalidate @submit.prevent="onSubmit">
    <div v-if="serverError" role="alert" class="form-error-server">
      {{ serverError }}
    </div>

    <div class="form-field">
      <label for="email">Email</label>
      <input
        id="email"
        v-model="email"
        v-bind="emailAttrs"
        type="email"
        autocomplete="email"
        :aria-invalid="!!errors.email"
        :aria-describedby="errors.email ? 'email-error' : undefined"
      />
      <span v-if="errors.email" id="email-error" role="alert" class="form-field-error">
        {{ errors.email }}
      </span>
    </div>

    <div class="form-field">
      <label for="password">Mot de passe</label>
      <input
        id="password"
        v-model="password"
        v-bind="passwordAttrs"
        type="password"
        autocomplete="current-password"
        :aria-invalid="!!errors.password"
        :aria-describedby="errors.password ? 'password-error' : undefined"
      />
      <span v-if="errors.password" id="password-error" role="alert" class="form-field-error">
        {{ errors.password }}
      </span>
    </div>

    <div class="form-actions">
      <button type="button" :disabled="isSubmitting" @click="emit('cancel')">
        Annuler
      </button>
      <button type="submit" :disabled="isSubmitting">
        <span v-if="isSubmitting">Connexion...</span>
        <span v-else>Se connecter</span>
      </button>
    </div>
  </form>
</template>
```

**Règles obligatoires :**
- `novalidate` sur le `<form>` — VeeValidate gère la validation, pas le navigateur
- Attributs ARIA (`aria-invalid`, `aria-describedby`) sur chaque champ avec erreur
- `role="alert"` sur les messages d'erreur (champ et serveur)
- `autocomplete` sur les champs email/password pour l'accessibilité
- `isSubmitting` local (pas le `isSubmitting` de VeeValidate) pour gérer l'état async post-validation
- Erreurs serveur séparées des erreurs de validation — affichées en haut du formulaire

---

## Étape 4 — Vérification TypeScript

```bash
npx vue-tsc --noEmit 2>&1 | head -30
```

---

## Résultat attendu

Rapporter :
- Fichiers créés (composant + schéma)
- Champs du formulaire et règles de validation
- Événements emits définis
- Erreurs TypeScript détectées
