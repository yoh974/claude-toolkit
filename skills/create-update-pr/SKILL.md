---
name: create-update-pr
description: "Create or update an Azure DevOps PR with full description, work item link and reviewers for the platform repos"
argument-hint: "[pr-title?] e.g. 'fix(68387): fix questionnaire list not updated'"
allowed-tools: ["Bash", "Read", "Glob", "Grep", "Agent"]
---

# create-update-pr — Create or Update a Pull Request

**Syntaxe :** `/create-update-pr [pr-title?]`

- `pr-title` : optional custom title (defaults to last commit message)

**Arguments :** "$ARGUMENTS"

## Context

- **Azure DevOps org**: https://dev.azure.com/your-org — project: TeamArea
- **Target branch**: always `develop`
- **User team**: Core Team

### Core Team dev reviewers (always add, no approval needed)
- Reviewer A — reviewer-a@example.com
- Reviewer B — reviewer-b@example.com
- Reviewer C — reviewer-c@example.com (à ne pas ajouter si absent pour le moment)
- Reviewer D — reviewer-d@example.com
- Reviewer E — reviewer-e@example.com

### Repo-specific reviewers (⚠️ ALWAYS ask user approval before adding)
| Repository | Reviewer | Email |
|---|---|---|
| apps/medical-visit-web | Reviewer One | reviewer1@example.com |
| services/medical-visit-service | Reviewer Two | (ask user for email if unknown) |

---

## Step 1 — Gather branch and repo context

```bash
# Current branch, repo name, last commit
git branch --show-current
git log develop..HEAD --oneline
git diff develop...HEAD --stat
git remote get-url origin
```

Extract from branch name or last commit message:
- **Work item ID**: e.g. `fix/68387-...` → `68387`
- **Repo name**: from git remote URL (e.g. `apps/medical-visit-web`)

---

## Step 2 — Check for existing PR

```bash
BRANCH=$(git branch --show-current)
az repos pr list \
  --repository {repo-name} \
  --source-branch "$BRANCH" \
  --status active \
  --output table
```

- If a PR exists → note its ID, proceed to **update** it
- If no PR → proceed to **create** it

---

## Step 3 — Read the diff to understand the changes

```bash
git diff develop...HEAD
```

Analyze the diff to write:
1. **Problème** — what was broken or missing before this change
2. **Solution** — what was implemented and why it works
3. **Impact / régressions possibles** — files or behaviors that could be affected by this change (1-3 bullet points max, omit if none)

---

## Step 3.5 — Detect new Kafka topics

New topics can come from `topic.ts` files (domain topics) **or** from `*.contract.ts` files (command/event topics). Run both checks:

```bash
# Domain topics (topic.ts files)
git diff develop...HEAD -- "*/kafka-topics/**/topic.ts" | grep "^+" | grep -v "^+++" | grep "Name:"

# Command/event topics (contract files)
git diff develop...HEAD -- "*/kafka-topics/**/*.contract.ts" | grep "^+" | grep -v "^+++" | grep "Name:"
```

For each line found, extract the resolved topic name by replacing `${BaseTopic}` with its value. To get `BaseTopic`:

```bash
grep -r "BaseTopic" src/core/kafka/kafka-topics/<domain>/v1alpha/base.topic.ts 2>/dev/null | head -5
```

Typically `BaseTopic = "services/patient-record-service.v1alpha"`.

**Domain topics** (`topic.ts`): each `Name:` entry generates **two** topics to transmit:
- the topic itself (e.g. `services/patient-record-service.v1alpha.domain.MyAggregate`)
- its dead-letter topic (append `.dlt`)

Also check for new `DeadLetterTopic:` lines to confirm the DLT name if it differs from the default pattern.

**Command/event topics** (`*.contract.ts`): each `Name:` entry is **one** topic (no DLT). Exclude `ReplyTopic` lines — do not include reply topics.

---

## Step 3.6 — Build the PR description

Write the description in **French**, using this markdown structure.

If new Kafka topics were detected in step 3.5, **prepend** the topics section before "Problème". List each topic **one per line, no bullet points, no backticks**:

```
## Topics à transmettre aux devops

services/patient-record-service.v1alpha.domain.MyAggregate
services/patient-record-service.v1alpha.domain.MyAggregate.dlt
services/patient-record-service.v1alpha.command.AddMyEntity
...
```

Full description template:

```
## Topics à transmettre aux devops        ← only if new topics detected

<topic-name>
<topic-name>.dlt
...

## Problème

<explanation of the bug or missing feature>

## Solution

<explanation of what was changed and how it fixes the problem>

## Impact / régressions possibles

- <impacted area or behavior — omit section entirely if no notable risk>
```

If no new Kafka topics were detected, omit the "Topics à transmettre aux devops" section entirely.

---

## Step 4 — Determine PR title

- If `$ARGUMENTS` is non-empty → use it as title
- Otherwise → use the last commit message subject

---

## Step 5 — Create or update the PR

### If creating:
```bash
az repos pr create \
  --repository {repo-name} \
  --source-branch {branch} \
  --target-branch develop \
  --title "{title}" \
  --description "{description}"
```

Note the new PR ID from the output.

### If updating:
```bash
az repos pr update \
  --id {pr-id} \
  --title "{title}" \
  --description "{description}"
```

---

## Step 6 — Link work item

If a work item ID was found in step 1:

```bash
az repos pr work-item add --id {pr-id} --work-items {work-item-id}
```

If no work item ID could be extracted, ask the user: "Quel est l'ID du work item à lier ?"

---

## Step 7 — Add Core Team dev reviewers

```bash
az repos pr reviewer add \
  --id {pr-id} \
  --reviewers reviewer-a@example.com reviewer-b@example.com reviewer-c@example.com
```

Skip any reviewer who is already on the PR (check the reviewer list from step 2 output).

---

## Step 8 — Ask about repo-specific reviewer

Check the repo name from step 1 against the table above.

- **apps/medical-visit-web** → ask: "Voulez-vous ajouter **Reviewer One** (reviewer1@example.com) comme reviewer ?"
- **services/medical-visit-service** → ask: "Voulez-vous ajouter **Reviewer Two** comme reviewer ?"

If the user confirms, add them:
```bash
az repos pr reviewer add --id {pr-id} --reviewers {email}
```

Do NOT add them without explicit confirmation.

---

## Step 8.5 — Set work item state to "In Review"

If a work item ID was found in step 1:

```bash
az boards work-item update --id {work-item-id} --state "In Review"
```

---

## Step 9 — Report

Print a summary:

```
✓ PR #{id} — {title}
  → {url}
  → Work item: #{work-item-id}
  → Reviewers: Reviewer A, Reviewer B, Reviewer C, Reviewer D, Reviewer E [+ repo-specific if added]
```
