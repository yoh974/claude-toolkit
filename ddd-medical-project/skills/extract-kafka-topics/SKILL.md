---
name: extract-kafka-topics
description: "Extrait les topics Kafka nouveaux/modifiés depuis la diff git (branche vs develop ou depuis un commit) et génère un ticket Azure DevOps markdown"
argument-hint: "[<commit-sha>] — SHA de commit optionnel comme point de comparaison (défaut: diff vs develop)"
allowed-tools: ["Bash", "Read", "Grep"]
---

# Extract Kafka Topics — Ticket Azure DevOps

Génère la liste des topics Kafka à provisionner, basée **uniquement sur les fichiers modifiés** dans cette branche.

**Arguments:** "$ARGUMENTS"

Si un SHA de commit est fourni, utiliser `git diff <sha>...HEAD`. Sinon, utiliser `git diff develop...HEAD`.

---

## Étape 1 — Obtenir les fichiers de topics modifiés

```bash
# Sans argument : diff vs develop
git diff develop...HEAD --name-only -- "src/core/kafka/kafka-topics/"

# Avec SHA : git diff <sha>...HEAD --name-only -- "src/core/kafka/kafka-topics/"
```

Filtrer uniquement les fichiers `topic.ts` ou `topics.ts` (pas les contracts, pas les index).

---

## Étape 2 — Obtenir la diff complète de ces fichiers

```bash
git diff develop...HEAD -- src/core/kafka/kafka-topics/ -- "*.ts"
```

Dans la diff, isoler les **lignes ajoutées** (`+`) contenant `Name:`, `DeadLetterTopic:`, `ReplyTopic:`.

---

## Étape 3 — Résoudre les BaseTopic

Pour chaque fichier topic modifié, lire le `base.topic.ts` du même namespace :

```bash
# Exemple pour un fichier dans connector/v1alpha/
cat src/core/kafka/kafka-topics/connector/v1alpha/base.topic.ts
```

Extraire `Namespace` et `ApiVersion`, construire `BaseTopic = Namespace + '.' + ApiVersion`.

Substituer `${BaseTopic}` dans chaque ligne extraite pour obtenir le nom complet du topic.

---

## Étape 4 — Générer la liste brute

Collecter tous les noms résolus (Name, DeadLetterTopic, ReplyTopic) dans l'ordre où ils apparaissent dans la diff.

## Sortie

Afficher **uniquement** la liste des topics, un par ligne, sans espace, sans puce, sans backtick, sans markdown :

```
services/legacy-connector-service.v1alpha.message.WorkIncapacitySituation
services/legacy-connector-service.v1alpha.message.WorkIncapacitySituation.dlt
services/patient-record-service.v1alpha.command.ImportWorkIncapacity
services/patient-record-service.v1alpha.command.ImportWorkIncapacity.reply
```

Rien d'autre. Pas de titre, pas d'explication, pas de récapitulatif.
