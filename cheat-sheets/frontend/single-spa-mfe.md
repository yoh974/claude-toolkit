# Single-SPA / Module Federation (Micro-Frontends)

Plusieurs micro-frontends Angular chargés dynamiquement dans un shell hôte via single-spa + Webpack Module Federation.

## Double point d'entrée

- **Standalone** : `src/main.ts` → `provideRouter(routes)`, lancé seul pour le dev
- **MFE** : `src/main.single-spa.ts` → lifecycle hooks single-spa, exposé via Module Federation

```ts
// webpack config du MFE
exposes: { './Root': './src/main.single-spa.ts' }
// output: remoteEntry.js
```

Seule dépendance partagée en singleton côté shell : `zone.js`.

## Le shell (host)

- Contexte global partagé : `window.<app>_root.context` (rôles, permissions, état cross-MFE)
- Gère l'échange de token OIDC, le routing inter-MFE (`single-spa-layout` dans `index.ejs`)
- Émet des events `mfe:state` pour déclencher les pages d'erreur (403/404/500)
- Résolution des remotes en prod : `https://<host>/mfe/<mfe>/remoteEntry.js`

## Commandes

```bash
npm run start:single-spa       # un MFE seul, mode intégré
make serve-microfrontends      # sélecteur interactif (shell toujours démarré)
make serve-all-mfe             # tous les MFE + shell
make serve-shell-only          # shell seul
```

## Pièges courants

- Déclarer une dépendance partagée (autre que `zone.js`) sans la marquer `singleton: true` côté shell → double instance Angular, bugs de change detection fantômes
- Oublier de lire le token depuis `localStorage` injecté par le shell → le MFE pense être déconnecté
- Lancer un MFE en standalone (`main.ts`) en pensant tester l'intégration shell — utiliser `start:single-spa` pour ça
