---
name: majestic-avalanche
description: "Orchestrateur du workflow de développement automatisé. Reçoit le contenu d'un template CLAUDE_TEMPLATE, structure le plan via CCOA, pilote les agents de dev/test/doc/git et crée la PR."
argument-hint: "<contenu_template_claude>"
allowed-tools: ["Bash", "Agent", "Read", "Glob", "Grep"]
---

# majestic-avalanche — Pipeline de développement automatisé

**Syntaxe :** `/majestic-avalanche <contenu_template>`

**Arguments :** "$ARGUMENTS"

---

## Mapping repos → répertoires et agents

| Code template | Repo git                         | Répertoire (relatif à workspace) | Agent          |
|---------------|----------------------------------|-------------------------------------|----------------|
| QT.back       | services/questionnaire-service    | ../services/questionnaire-service    | nestjs-dev-back |
| IR.back       | services/patient-record-service               | ../services/patient-record-service               | nestjs-dev-back |
| MV.back       | services/medical-visit-service               | ../services/medical-visit-service               | nestjs-dev-back |
| RF.back       | services/reference-data-service      | ../services/reference-data-service      | nestjs-dev-back |
| VS.back       | services/teleconsultation-service            | ../services/teleconsultation-service            | nestjs-dev-back |
| QT.front      | apps/questionnaire-web | ../apps/questionnaire-web | angular-dev-front |
| IR.front      | apps/patient-record-web            | ../apps/patient-record-web            | angular-dev-front |
| MV.front      | apps/medical-visit-web            | ../apps/medical-visit-web            | angular-dev-front |

---

## Étape 1 — Parser et afficher le résumé du template

Extraire les champs du template `$ARGUMENTS` :

```
repos:       liste des repos concernés (ex: QT.back, QT.front)
context:     contexte métier/technique
actions:     liste des actions à réaliser
constraints: contraintes
objectives:  objectif attendu
examples:    (optionnel)
unit-tests:  oui | non
key-files:   fichiers/dossiers clés
```

Afficher un résumé :
```
📋 Template CLAUDE détecté
   Repos    : QT.back, QT.front
   Tests    : oui
   Fichiers : src/modules/...
   Objectif : [résumé 1 ligne]
```

---

## Étape 2 — Structurer le plan avec CCOA

Invoquer le skill `ccoa` en lui passant le contenu complet du template comme prompt.

CCOA va structurer le travail en : Contexte, Contraintes, Objectif, Actions — et entrer en mode plan.

**Attendre la validation du plan par l'utilisateur avant de passer à l'étape 3.**

---

## Étape 3 — Déléguer au token-optimizer

Une fois le plan CCOA validé, invoquer l'agent `token-optimizer` en lui fournissant :

```
- Plan CCOA structuré (contexte + contraintes + objectifs + actions)
- Repos cibles avec mapping : [{code, repo, répertoire, agent}]
- unit-tests: oui|non
- key-files: [liste]
- Branch courante (git branch --show-current)
- ID de la tâche ADO (extrait du nom de branche si possible)
```

L'agent `token-optimizer` est responsable de :
1. Enchaîner les agents de développement (nestjs-dev-back / angular-dev-front)
2. Lancer nestjs-test si unit-tests = oui
3. Lancer doc-writer pour compléter la documentation existante
4. Lancer git-ops pour les commits atomiques et le push
5. Surveiller l'usage tokens et déclencher la procédure d'urgence à 98%

---

## Étape 4 — Créer la Pull Request

Quand token-optimizer signale que le travail est terminé (dev + tests + doc + push), invoquer le skill `create-update-pr`.

---

## Étape 5 — Résumé final

```
✅ Workflow majestic-avalanche terminé
   Branch  : {nom-branche}
   Commits : {n} commits
   PR      : #{id} — {url}
   Tests   : {oui|non — n specs}
   Doc     : {modifié|inchangé}
```

---

## Gestion des erreurs

- **Repo inconnu dans le mapping** : demander à l'utilisateur le répertoire cible avant de continuer
- **Compilation échouée** : stopper, afficher l'erreur, attendre instruction utilisateur
- **Push rejeté** : signaler le conflit, ne pas forcer
- **Limite tokens atteinte (98%)** : token-optimizer déclenche commit+push d'urgence + commentaire ADO, arrêter le workflow proprement
