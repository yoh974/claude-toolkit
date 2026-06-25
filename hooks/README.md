# Claude Code Hooks

Les **hooks** sont des commandes shell exécutées automatiquement à des moments précis du cycle de vie de Claude Code (avant/après un outil, à l'ouverture d'une session, sur notification…). Ils permettent d'ajouter des garde-fous, de l'automatisation et de l'observabilité sans modifier le modèle.

Configuration dans un fichier `settings.json` :

| Fichier | Portée | Git |
|---|---|---|
| `~/.claude/settings.json` | Globale (tous projets) | — |
| `.claude/settings.json` | Projet (partagé équipe) | committé |
| `.claude/settings.local.json` | Projet (perso) | gitignoré |

Voir [`settings.example.json`](settings.example.json) pour une config complète prête à l'emploi.

## Événements principaux

| Événement | Matcher | Quand | Usage type |
|---|---|---|---|
| `PreToolUse` | nom d'outil | avant un outil (peut bloquer) | garde-fou, log, réécriture d'arguments |
| `PostToolUse` | nom d'outil | après un outil réussi | formatage, lint, notification |
| `Notification` | — | sur notification de Claude | notification bureau |
| `SessionStart` | — | à l'ouverture d'une session | injection de contexte (git, env) |
| `Stop` | — | quand Claude termine son tour | récap, stats tokens |
| `PreCompact` / `PostCompact` | `manual`/`auto` | avant/après compactage | sauvegarde de contexte |
| `UserPromptSubmit` | — | à l'envoi d'un prompt | validation, enrichissement |

Matchers courants : `Bash`, `Write`, `Edit`, `Read`, `Glob`, `Grep`. Combinables en regex : `"Write|Edit"`, `"mcp__.*"`.

> Les hooks s'exécutent **même** sous `--dangerously-skip-permissions` / `bypassPermissions` : ils restent le dernier filet de sécurité.

## Structure

```json
{
  "hooks": {
    "EVENT": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "…", "timeout": 30 }
        ]
      }
    ]
  }
}
```

Types de hook : `command` (shell), `prompt` (évaluation LLM), `agent` (agent outillé), `http`, `mcp_tool`.

## Entrée / sortie

Le hook reçoit un **JSON sur stdin** (pas en variables d'env) :

```json
{
  "session_id": "abc123",
  "tool_name": "Write",
  "tool_input": { "file_path": "/path/to/file", "content": "…" },
  "tool_response": { "success": true }
}
```

Extraire les champs avec `jq -r`. Le hook peut renvoyer du JSON pour piloter le comportement :

- `systemMessage` — message affiché à l'utilisateur
- `continue: false` + `stopReason` — bloque l'action
- `hookSpecificOutput.additionalContext` — texte injecté dans le contexte du modèle
- `hookSpecificOutput.permissionDecision` — `allow` / `deny` / `ask` (PreToolUse)

## Les 4 hooks de l'exemple

### PreToolUse — log des commandes Bash
```bash
jq -r '"\(now|todate) \(.tool_input.command)"' >> ~/.claude/bash-history.log 2>/dev/null || true
```
Journalise chaque commande Bash horodatée dans `~/.claude/bash-history.log`. Ne bloque jamais (`|| true`).

### PostToolUse — notification d'édition (`Write|Edit`)
```bash
jq -r '.tool_input.file_path // .tool_response.filePath' | { read -r f; notify-send "Claude Code" "Modifié: $f"; } 2>/dev/null || true
```
Notification bureau à chaque fichier écrit/modifié. Linux (`notify-send`).

### Notification — popup bureau
```bash
jq -r '.message' | { read -r m; notify-send "Claude Code" "$m"; } 2>/dev/null || true
```
Relaie les notifications de Claude (permission en attente, tâche finie…) vers le bureau.

### SessionStart — injection du contexte git
```bash
git status -sb 2>/dev/null | jq -Rs '{hookSpecificOutput:{hookEventName:"SessionStart",additionalContext:("Git status:\n"+.)}}' 2>/dev/null || true
```
Injecte l'état git du dépôt dans le contexte de la session au démarrage.

## Hooks fournis par un plugin (exemple : caveman)

Un plugin peut **livrer ses propres hooks** : pas besoin de les écrire dans `settings.json`, ils sont déclarés dans le manifeste du plugin et chargés automatiquement quand le plugin est activé (`enabledPlugins`). Le chemin des scripts utilise le placeholder `${CLAUDE_PLUGIN_ROOT}`, résolu vers la racine du plugin.

Le plugin [caveman](https://github.com/JuliusBrussee/caveman) (mode de communication ultra-compressé) déclare deux hooks :

```json
{
  "SessionStart": [
    {
      "hooks": [
        {
          "type": "command",
          "command": "node \"${CLAUDE_PLUGIN_ROOT}/src/hooks/caveman-activate.js\"",
          "timeout": 5,
          "statusMessage": "Loading caveman mode..."
        }
      ]
    }
  ],
  "UserPromptSubmit": [
    {
      "hooks": [
        {
          "type": "command",
          "command": "node \"${CLAUDE_PLUGIN_ROOT}/src/hooks/caveman-mode-tracker.js\"",
          "timeout": 5,
          "statusMessage": "Tracking caveman mode..."
        }
      ]
    }
  ]
}
```

- `SessionStart` → `caveman-activate.js` : active le mode et injecte les règles de style au démarrage.
- `UserPromptSubmit` → `caveman-mode-tracker.js` : ré-injecte un rappel du mode à chaque prompt et suit le niveau actif (`lite`/`full`/`ultra`).

Le plugin fournit aussi un **statusline** (badge `[CAVEMAN]`) à activer côté `settings.json` :

```json
"statusLine": {
  "type": "command",
  "command": "bash \"${CLAUDE_PLUGIN_ROOT}/src/hooks/caveman-statusline.sh\""
}
```

> À retenir : pour un comportement réutilisable et versionné, un **plugin** (hooks + skills + commandes packagés) est préférable à des hooks épars dans `settings.json`. Les hooks de plugin et ceux de `settings.json` coexistent et se cumulent sur un même événement.

## Permissions

```json
"permissions": {
  "defaultMode": "bypassPermissions",
  "deny": ["Bash(rm -rf *)", "Bash(sudo *)", "Bash(git push --force *)"]
}
```

⚠️ **`bypassPermissions` supprime tous les prompts de permission.** La liste `deny` devient la seule barrière — gardez-y au minimum les commandes destructrices. Pour revenir au comportement normal, mettre `defaultMode` à `"default"`.

## Pré-requis

- `jq` — parsing du JSON d'entrée
- `notify-send` (paquet `libnotify-bin`) — notifications bureau Linux ; à adapter sur macOS (`osascript -e 'display notification …'`)

## Application

Copier le contenu de `settings.example.json` dans `~/.claude/settings.json` (fusionner avec l'existant, ne pas écraser). Les nouveaux hooks ne sont pas chargés à chaud : ouvrir `/hooks` une fois ou redémarrer Claude Code. Vérifier/éditer/désactiver ensuite via `/hooks`.
