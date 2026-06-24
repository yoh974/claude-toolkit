---
name: start-task
description: "Trouve les tâches actives Azure DevOps assignées à l'utilisateur courant, détecte les tâches avec template CLAUDE, vérifie processedByClaude, crée la branche et démarre le workflow majestic-avalanche si template présent."
argument-hint: "Aucun argument requis"
allowed-tools: ["Bash", "Agent"]
---

# start-task — Démarrer une tâche de l'itération courante

**Syntaxe :** `/start-task`

## Contexte Azure DevOps

- **Org** : https://dev.azure.com/your-org
- **Project** : TeamArea
- **Utilisateur** : Dev Name → `dev@example.com`

---

## Étape 1 — Récupérer les tâches actives assignées

```bash
az boards query \
  --wiql "SELECT [System.Id], [System.Title], [System.State], [System.WorkItemType], [System.Parent] FROM WorkItems WHERE [System.WorkItemType] = 'Task' AND [System.AssignedTo] = 'dev@example.com' AND [System.State] IN ('Active', 'In Progress', 'To Do') AND [System.IterationPath] UNDER @currentIteration('[TeamArea]\\Sentinels') ORDER BY [System.Id] DESC" \
  --org https://dev.azure.com/your-org \
  --project "TeamArea" \
  --output json
```

Si `@currentIteration` n'est pas supporté :
```bash
az boards query \
  --wiql "SELECT [System.Id], [System.Title], [System.State], [System.Parent] FROM WorkItems WHERE [System.WorkItemType] = 'Task' AND [System.AssignedTo] = 'dev@example.com' AND [System.State] NOT IN ('Done', 'Closed', 'Removed') ORDER BY [System.ChangedDate] DESC" \
  --org https://dev.azure.com/your-org \
  --project "TeamArea" \
  --output json
```

---

## Étape 2 — Analyser chaque tâche : description + commentaires

Pour chaque tâche récupérée, exécuter en parallèle :

```bash
# Lire description et commentaires de la tâche
az boards work-item show \
  --id {TASK_ID} \
  --org https://dev.azure.com/your-org \
  --output json | jq '{id: .id, title: .fields["System.Title"], description: .fields["System.Description"]}'

az boards work-item comment list \
  --id {TASK_ID} \
  --org https://dev.azure.com/your-org \
  --project "TeamArea" \
  --output json | jq '[.comments[].text]'
```

**Détection template** : la description contient `## CLAUDE_TEMPLATE` → tâche éligible au workflow automatique.

**Détection processedByClaude** : un commentaire contient le texte `processedByClaude` → tâche déjà en cours de traitement, **ne pas retraiter**.

---

## Étape 3 — Présenter la liste à l'utilisateur

Afficher les tâches en deux groupes :

```
🤖 Tâches avec template CLAUDE (workflow automatique)
ID      | Titre                          | État
--------|--------------------------------|----------
12345   | Corriger le bug author QT      | Active

📋 Tâches sans template (workflow manuel)
ID      | Titre                          | État
--------|--------------------------------|----------
12346   | Autre tâche                    | To Do
```

- Les tâches avec `processedByClaude` sont signalées `⏳ en cours` et ne sont pas proposées.
- Utiliser `AskUserQuestion` avec max 4 options (priorité aux tâches avec template).

---

## Étape 4 — Récupérer le ticket parent et son type

```bash
PARENT_ID=$(az boards work-item show --id {TASK_ID} --org https://dev.azure.com/your-org --output json | jq -r '.fields["System.Parent"]')

az boards work-item show \
  --id $PARENT_ID \
  --org https://dev.azure.com/your-org \
  --output json | jq '{id: .id, title: .fields["System.Title"], type: .fields["System.WorkItemType"]}'
```

---

## Étape 5 — Déterminer le préfixe de branche

| Type du parent      | Préfixe |
|---------------------|---------|
| User Story          | `feat`  |
| Bug                 | `fix`   |
| Backlog Item (PBI)  | `chore` |
| Feature             | `feat`  |
| Epic                | `chore` |
| *Autre / inconnu*   | `chore` |

---

## Étape 6 — Construire le nom de branche

Format : `{préfixe}/{TASK_ID}-{slug-du-titre}`

Règles slug :
1. Minuscules
2. Diacritiques remplacés (é→e, è→e, ê→e, à→a, ù→u, ô→o, î→i, ç→c…)
3. Espaces et caractères spéciaux → `-`
4. Tirets multiples → tiret simple
5. Max 50 chars pour la partie slug
6. Titre toujours en anglais

---

## Étape 7 — Créer la branche depuis develop

```bash
git checkout develop
git pull origin develop
git checkout -b {nom-de-branche}
```

---

## Étape 8 — Si la tâche a un template : marquer et déclencher le workflow

### 8a. Ajouter le commentaire processedByClaude

```bash
az boards work-item comment add \
  --id {TASK_ID} \
  --org https://dev.azure.com/your-org \
  --project "TeamArea" \
  --text "processedByClaude — $(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

### 8b. Extraire le contenu du template

Extraire depuis la description tout le bloc compris entre `## CLAUDE_TEMPLATE` et la prochaine section `##` (ou fin de texte). Nettoyer le HTML éventuel (Azure DevOps stocke parfois en HTML).

### 8c. Invoquer le workflow majestic-avalanche

Appeler le skill `majestic-avalanche` en lui passant le contenu brut du template comme argument :

```
/majestic-avalanche {contenu_template}
```

---

## Étape 9 — Confirmation finale (tâches sans template)

Pour les tâches sans template, afficher simplement :

```
✅ Branche créée : feat/12345-implement-contact-form
   Tâche : #12345 — Implémenter le formulaire de contact
   Parent : #9876 — [User Story] Gestion du formulaire de contact
```

---

## Gestion des erreurs

- **Aucune tâche trouvée** : afficher message + proposer `/azure-devops-cli`
- **Parent introuvable** : utiliser `chore` par défaut
- **Branche déjà existante** : proposer `git checkout {branche}`
- **develop pas à jour** : signaler les divergences
- **processedByClaude détecté** : afficher `⏳ Tâche #ID déjà en cours (processedByClaude). Choisissez une autre tâche.`
