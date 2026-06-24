---
name: import-recovery-data
description: "Insère directement des RecoveryData en MongoDB (1 item par type : SOCIAL, HEALTH, WORK, IDENTITY) pour un individu IR, en bypassant le pipeline ir_import. Par défaut: dernier individu créé."
argument-hint: "[individualRecordId?]"
allowed-tools: ["Bash"]
---

# import-recovery-datas — Seed recovery datas (insertion directe MongoDB)

**Syntaxe :** `/import-recovery-datas [individualRecordId?]`

**Arguments :** "$ARGUMENTS"

## Étape 1 — Résoudre l'individu

```bash
ARGS="$ARGUMENTS"
IND_ARG=$(echo "$ARGS" | awk '{print $1}')
IR_DB="ir-module"
BASE_URL="http://ir.api.localhost/api/v1/individual-records"
NOW=$(date -u +"%Y-%m-%dT%H:%M:%S.000Z")

if [ -z "$IND_ARG" ]; then
  IND=$(docker exec workspace-mongodb-1 mongosh --quiet --eval "
    db = db.getSiblingDB('$IR_DB');
    var ev = db.events.findOne(
      {aggregate_type:'Identity'},
      {'payload.individualRecordId': 1},
      {sort: {created_at: -1}}
    );
    print(ev && ev.payload ? ev.payload.individualRecordId : 'NOT_FOUND');
  " 2>/dev/null | tail -1 | tr -d '\r')
else
  IND="$IND_ARG"
fi

if [ "$IND" = "NOT_FOUND" ] || [ -z "$IND" ]; then
  echo "ERREUR: individualRecordId introuvable. Passez-le en argument."
  exit 1
fi

echo "Individu : $IND"
```

## Étape 2 — Vérifier si un recovery_data existe déjà

```bash
EXISTING_RD=$(docker exec workspace-mongodb-1 mongosh --quiet --eval "
  db = db.getSiblingDB('$IR_DB');
  var rd = db.recovery_datas.findOne({individualRecordId: '$IND'}, {_id:1});
  print(rd ? rd._id : 'NONE');
" 2>/dev/null | tail -1 | tr -d '\r')

if [ "$EXISTING_RD" != "NONE" ] && [ -n "$EXISTING_RD" ]; then
  echo "RecoveryData existant : $EXISTING_RD"
  echo "Ajout d'items aux types manquants..."
  UPSERT_MODE="update"
else
  echo "Aucun RecoveryData existant — création."
  UPSERT_MODE="insert"
fi
```

## Étape 3 — Générer les UUIDs côté shell

```bash
RD_ID=$(cat /proc/sys/kernel/random/uuid)
ITEM_SOCIAL=$(cat /proc/sys/kernel/random/uuid)
ITEM_HEALTH=$(cat /proc/sys/kernel/random/uuid)
ITEM_WORK=$(cat /proc/sys/kernel/random/uuid)
ITEM_IDENTITY=$(cat /proc/sys/kernel/random/uuid)
```

## Étape 4 — Insérer ou mettre à jour la projection recovery_datas

```bash
echo ""
echo "=== Insertion directe en MongoDB ==="

docker exec workspace-mongodb-1 mongosh --quiet --eval "
  db = db.getSiblingDB('$IR_DB');
  var NOW = new Date('$NOW');
  var IND = '$IND';
  var traceability = { author: 'seed-import', date: NOW, sourceContext: 'LEGACY' };

  var newItems = [
    {
      id: '$ITEM_SOCIAL',
      individualRecordId: IND,
      title: 'Maladie professionnelle - Tableau 57',
      type: 'SOCIAL',
      category: 'Maladie professionnelle',
      traceability: traceability,
      fields: [
        { name: 'diseaseName',       label: 'Désignation de la maladie',  value: 'Affections périarticulaires - Tableau 57', type: 'string' },
        { name: 'declarationDate',   label: 'Date de déclaration',        value: '2023-10-01', type: 'date'   },
        { name: 'recognitionStatus', label: 'Statut de reconnaissance',   value: 'ACCEPTED',   type: 'string' },
        { name: 'employer',          label: 'Employeur concerné',         value: 'Entreprise Industrielle SA', type: 'string' },
        { name: 'incapacityRate',    label: 'Taux incapacite (%)',        value: 15,           type: 'int'    },
        { name: 'consolidationDate', label: 'Date de consolidation',      value: '2024-03-15', type: 'date'   }
      ]
    },
    {
      id: '$ITEM_HEALTH',
      individualRecordId: IND,
      title: 'Lombalgie chronique',
      type: 'HEALTH',
      category: 'Pathologie',
      traceability: traceability,
      fields: [
        { name: 'diagnosisDate', label: 'Date de diagnostic', value: '2022-06-15', type: 'date'   },
        { name: 'pathologyName', label: 'Pathologie',          value: 'Lombalgie chronique professionnelle', type: 'string' },
        { name: 'isChronicPain', label: 'Douleur chronique',   value: true,        type: 'bool'   },
        { name: 'icd10',         label: 'Code CIM-10',         value: 'M54.5',     type: 'string' }
      ]
    },
    {
      id: '$ITEM_WORK',
      individualRecordId: IND,
      title: 'Restriction port de charges',
      type: 'WORK',
      category: 'Restriction',
      traceability: traceability,
      fields: [
        { name: 'startDate',       label: 'Date de début',       value: '2024-01-08', type: 'date'   },
        { name: 'restrictionType', label: 'Type de restriction', value: 'Port de charges > 5 kg interdit', type: 'string' },
        { name: 'prescriber',      label: 'Prescripteur',        value: 'Dr. Martin - Médecin du travail', type: 'string' }
      ]
    },
    {
      id: '$ITEM_IDENTITY',
      individualRecordId: IND,
      title: 'Adresse legacy',
      type: 'IDENTITY',
      category: 'Coordonnées',
      traceability: traceability,
      fields: [
        { name: 'street',  label: 'Rue',        value: '12 rue des Acacias', type: 'string' },
        { name: 'city',    label: 'Ville',       value: 'Lyon',              type: 'string' },
        { name: 'zipCode', label: 'Code postal', value: '69001',             type: 'string' }
      ]
    }
  ];

  var existing = db.recovery_datas.findOne({individualRecordId: IND});

  if (!existing) {
    db.recovery_datas.insertOne({
      _id: '$RD_ID',
      id: '$RD_ID',
      individualRecordId: IND,
      version: 4,
      createdAt: NOW,
      updatedAt: NOW,
      items: newItems
    });
    print('Created RecoveryData $RD_ID with 4 items (SOCIAL, HEALTH, WORK, IDENTITY)');
  } else {
    var existingTypes = existing.items.map(function(i) { return i.type; });
    var toAdd = newItems.filter(function(i) { return existingTypes.indexOf(i.type) === -1; });
    if (toAdd.length === 0) {
      print('All types already present — nothing to add.');
    } else {
      db.recovery_datas.updateOne(
        {_id: existing._id},
        { \$push: { items: { \$each: toAdd } }, \$set: { updatedAt: NOW }, \$inc: { version: toAdd.length } }
      );
      print('Added ' + toAdd.length + ' missing item(s): ' + toAdd.map(function(i){return i.type;}).join(', '));
    }
  }
" 2>/dev/null | tail -3
```

## Étape 5 — Vérification via l'API

```bash
echo ""
echo "=== Vérification API ==="

for TYPE in social health work identity; do
  COUNT=$(curl -s "$BASE_URL/$IND/recovery-datas/$TYPE" | python3 -c "import sys,json; d=json.load(sys.stdin); print(len(d.get('items',[])) if d else 0)" 2>/dev/null)
  echo "  $TYPE : $COUNT item(s)"
done

echo ""
echo "=== Terminé — Individu : $IND ==="
```