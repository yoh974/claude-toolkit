# Vue 3 Toolkit

Agents et skills prêts à l'emploi pour des projets **Vue 3/TypeScript** suivant les bonnes pratiques (Composition API, `<script setup>`, Pinia, Vue Router 4, Vite, VeeValidate + Zod).

Copiez les agents dans `.claude/agents/` et les hooks dans `.claude/settings.json` de votre projet.

---

## Agents

| Agent | Rôle | Modèle |
|---|---|---|
| `vue-dev` | Développeur Vue 3/TypeScript senior — composants SFC, composables, Pinia stores, Vue Router, API | sonnet |
| `vue-architect` | Expert architecture feature-first, Composition API avancée, performance, sécurité — ADR et conseils | sonnet |
| `vue-qa` | Ingénieur QA — tests Vitest, Vue Test Utils, Pinia Testing, Playwright | sonnet |
| `vue-reviewer` | Relecteur automatique déclenché au `git commit` — anti-patterns, TypeScript strict, XSS | sonnet |
| `vue-perf` | Expert performance — bundle size, lazy loading, virtualisation, Core Web Vitals | sonnet |

---

## Skills

| Skill | Usage |
|---|---|
| `/vue-component UserCard` | Crée un SFC Vue 3 avec props typées, emits, et composable associé optionnel |
| `/vue-composable useUserProfile User` | Crée un composable avec fetch, loading/error, cleanup `onUnmounted` |
| `/vue-store useCartStore CartItem` | Crée un Pinia setup store avec state, getters computed, actions async, `$reset()` |
| `/vue-page UserProfilePage /users/:id` | Crée une page Vue 3 + route lazy loading + guard de navigation si protégée |
| `/vue-form LoginForm email password` | Crée un formulaire avec VeeValidate + Zod, schéma typé, ARIA, erreurs serveur |

---

## Hook de review automatique

Le fichier `hooks/settings.example.json` contient un hook `PreToolUse` qui injecte le diff stagé (`.vue`, `.ts`, `.tsx`) dans le contexte de Claude juste avant un `git commit`, demandant au modèle d'invoquer l'agent `vue-reviewer`.

Fusionner avec `.claude/settings.json` ou `.claude/settings.local.json` de votre projet.

> Prérequis : `jq` installé, agent `vue-reviewer` présent dans `.claude/agents/`

---

## Stack recommandée

```
Vue 3.4+              Composition API, script setup, TypeScript
Vite 5+               Build tool, HMR, lazy chunks
Pinia 2+              Gestion d'état (setup store pattern)
Vue Router 4          Routing avec lazy loading et guards typés
VeeValidate 4 + Zod   Validation de formulaires
@vueuse/core          Composables utilitaires (optionnel mais recommandé)
Vitest + @vue/test-utils  Tests unitaires et de composants
Playwright            Tests E2E
```
