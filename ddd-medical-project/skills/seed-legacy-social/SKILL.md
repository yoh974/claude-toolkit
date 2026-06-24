---
name: seed-legacy-social
description: "Insère des données legacy simulées (maladie professionnelle + données de reprise) pour un individu IR. Poste une maladie professionnelle via HTTP et insère des RecoveryData directement en MongoDB. Par défaut: dernier individu créé."
argument-hint: "[individualRecordId?]"
allowed-tools: ["Bash"]
---

# seed-legacy-social — Seed données legacy social + reprise

**Syntaxe :** `/seed-legacy-social [individualRecordId?]`

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

## Étape 2 — Vérifier/créer l'agrégat Social

```bash
SOCIAL_EXISTS=$(docker exec workspace-mongodb-1 mongosh --quiet --eval "
  db = db.getSiblingDB('$IR_DB');
  var s = db.social.findOne({individualRecordId: '$IND'}, {_id:1});
  print(s ? 'yes' : 'no');
" 2>/dev/null | tail -1 | tr -d '\r')

if [ "$SOCIAL_EXISTS" != "yes" ]; then
  echo "Agrégat Social absent — création directe en MongoDB..."
  SOCIAL_ID=$(cat /proc/sys/kernel/random/uuid)
  EVENT_ID=$(cat /proc/sys/kernel/random/uuid)
  docker exec workspace-mongodb-1 mongosh --quiet --eval "
    db = db.getSiblingDB('$IR_DB');
    var NOW = new Date('$NOW');
    db.events.insertOne({
      _id: '$EVENT_ID', content_type: 'SocialCreated', created_at: NOW, created_by: 'system',
      aggregate_id: '$SOCIAL_ID', aggregate_type: 'Social', version: 1,
      metadata: {'x-request-id':null,'saga-context':null,'aggregate-id':'$SOCIAL_ID','aggregate-type':'Social','tenant-id':null,token:null,locale:null},
      payload: {id:'$SOCIAL_ID',individualRecordId:'$IND',lifeSituations:[],supportJourneys:[],economicsSituations:[],workAbsencesSituations:[],workAccidents:[],workIncapacities:[],workInvalidities:[],workRelatedDiseases:[],createdAt:'$NOW',updatedAt:'$NOW'}
    });
    db.social.insertOne({_id:'$SOCIAL_ID',id:'$SOCIAL_ID',individualRecordId:'$IND',version:1,createdAt:NOW,updatedAt:NOW});
    print('Social créé : $SOCIAL_ID');
  " 2>/dev/null | tail -2
else
  echo "Agrégat Social déjà présent."
fi
```

## Étape 3 — Résoudre un item référentiel

```bash
get_ref_item() {
  local PATTERN="$1"
  docker exec workspace-mongodb-1 mongosh --quiet --eval "
    db = db.getSiblingDB('$IR_DB');
    var item = db['referentiel.items'].findOne({code:{'\$regex':'$PATTERN','\$options':'i'}},{_id:1});
    if (!item) item = db['referentiel.items'].findOne({name:{'\$regex':'$PATTERN','\$options':'i'}},{_id:1});
    if (!item) item = db['referentiel.items'].findOne({},{_id:1});
    print(item ? item._id : '');
  " 2>/dev/null | tail -1 | tr -d '\r'
}
```

## Étape 4 — Poster la maladie professionnelle (HTTP)

```bash
echo "=== Maladie professionnelle (work-related-disease) ==="

MP_ID=$(get_ref_item "MALADIE|PROFES|TMS|BRUIT")
if [ -z "$MP_ID" ]; then
  echo "  SKIP: aucun item référentiel trouvé pour la maladie professionnelle"
else
  echo "  workRelatedDisease referentiel id: $MP_ID"
  curl -s -X POST "$BASE_URL/$IND/work-related-diseases" \
    -H "Content-Type: application/json" \
    -d "{
      \"workRelatedDisease\": \"$MP_ID\",
      \"companies\": [{\"text\": \"Entreprise Industrielle SA\"}],
      \"observationDate\": \"2023-09-15\",
      \"declarationDate\": \"2023-10-01\",
      \"declarationNumber\": 2023001,
      \"exposureRiskConcerns\": [{\"text\": \"Exposition aux vibrations mécaniques\"}],
      \"certificateDelivered\": true,
      \"certificateDeliveryFrame\": \"INTERNAL\",
      \"certificateHealthProfessionalFreeText\": \"Dr. Dupont\",
      \"relapse\": {\"occurred\": false},
      \"consolidation\": {\"occurred\": true, \"date\": \"2024-03-15\"},
      \"healing\": {\"occurred\": false},
      \"recognitionStatus\": \"ACCEPTED\"
    }" \
    | python3 -c "import sys,json;d=json.load(sys.stdin);print('  => workRelatedDiseaseId:', d.get('workRelatedDiseaseId', d.get('message','ERR')))" 2>/dev/null
fi
```

## Étape 5 — Insérer les données de reprise (RecoveryData) directement en MongoDB

```bash
echo ""
echo "=== Données de reprise (RecoveryData) ==="

# Vérifier si un RecoveryData existe déjà
RECOVERY_EXISTS=$(docker exec workspace-mongodb-1 mongosh --quiet --eval "
  db = db.getSiblingDB('$IR_DB');
  var r = db.recovery_datas.findOne({individualRecordId: '$IND'}, {_id:1});
  print(r ? r._id : 'NONE');
" 2>/dev/null | tail -1 | tr -d '\r')

if [ "$RECOVERY_EXISTS" = "NONE" ]; then
  RECOVERY_ID=$(cat /proc/sys/kernel/random/uuid)
  echo "  Création RecoveryData : $RECOVERY_ID"
else
  RECOVERY_ID="$RECOVERY_EXISTS"
  echo "  RecoveryData existant : $RECOVERY_ID"
fi

# Items de reprise legacy à insérer
docker exec workspace-mongodb-1 mongosh --quiet --eval "
  db = db.getSiblingDB('$IR_DB');
  var NOW = new Date('$NOW');
  var IND = '$IND';
  var RECOVERY_ID = '$RECOVERY_ID';

  var traceability = {
    author: 'legacy-import',
    date: NOW,
    sourceContext: 'LEGACY'
  };

  var items = [
    {
      id: UUID().toString(),
      individualRecordId: IND,
      title: 'Maladie professionnelle - Tableau 57',
      type: 'SOCIAL',
      category: 'Maladie professionnelle',
      traceability: traceability,
      fields: [
        { name: 'diseaseName',     label: 'Désignation de la maladie', value: 'Affections périarticulaires provoquées par certains gestes et postures de travail - Tableau 57', type: 'string' },
        { name: 'declarationDate', label: 'Date de déclaration',        value: '2023-10-01',  type: 'date'   },
        { name: 'recognitionDate', label: 'Date de reconnaissance',     value: '2024-01-15',  type: 'date'   },
        { name: 'recognitionStatus', label: 'Statut de reconnaissance', value: 'ACCEPTED',    type: 'string' },
        { name: 'employer',        label: 'Employeur concerné',         value: 'Entreprise Industrielle SA', type: 'string' },
        { name: 'exposureRisk',    label: 'Risque d\'exposition',       value: 'Vibrations mécaniques et postures contraignantes', type: 'string' },
        { name: 'incapacityRate',  label: 'Taux d\'incapacité (%)',     value: 15,            type: 'int'    },
        { name: 'consolidationDate', label: 'Date de consolidation',    value: '2024-03-15',  type: 'date'   }
      ]
    },
    {
      id: UUID().toString(),
      individualRecordId: IND,
      title: 'Accident du travail - Chute de hauteur',
      type: 'SOCIAL',
      category: 'Accident du travail',
      traceability: traceability,
      fields: [
        { name: 'accidentDate',    label: 'Date de l\'accident',        value: '2023-11-14',  type: 'date'   },
        { name: 'accidentType',    label: 'Type d\'accident',           value: 'Chute de hauteur (< 3 m)', type: 'string' },
        { name: 'employer',        label: 'Employeur',                  value: 'Entreprise Industrielle SA', type: 'string' },
        { name: 'workLinkRecognition', label: 'Lien avec le travail reconnu', value: true,    type: 'bool'   },
        { name: 'endDate',         label: 'Date de guérison/consolidation', value: '2023-12-20', type: 'date' },
        { name: 'sequelae',        label: 'Séquelles',                  value: 'Douleurs lombaires résiduelles', type: 'string' }
      ]
    },
    {
      id: UUID().toString(),
      individualRecordId: IND,
      title: 'Maintien dans l\'emploi - Suivi 2024',
      type: 'SOCIAL',
      category: 'Maintien dans l\'emploi',
      traceability: traceability,
      fields: [
        { name: 'startDate',       label: 'Date de début du suivi',     value: '2024-01-08',  type: 'date'   },
        { name: 'framework',       label: 'Cadre de prescription',      value: 'Interne (médecin du travail)', type: 'string' },
        { name: 'objective',       label: 'Objectif',                   value: 'Maintien dans l\'emploi suite à maladie professionnelle', type: 'string' },
        { name: 'hasProfessionalOrigin', label: 'Origine professionnelle', value: true,       type: 'bool'   },
        { name: 'prescriber',      label: 'Prescripteur',               value: 'Dr. Martin - Médecin du travail', type: 'string' },
        { name: 'status',          label: 'Statut du parcours',         value: 'En cours',    type: 'string' }
      ]
    },
    {
      id: UUID().toString(),
      individualRecordId: IND,
      title: 'Arrêt de travail - Lombalgie',
      type: 'SOCIAL',
      category: 'Absence',
      traceability: traceability,
      fields: [
        { name: 'startDate',       label: 'Date de début',              value: '2024-01-08',  type: 'date'   },
        { name: 'endDate',         label: 'Date de fin',                value: '2024-02-28',  type: 'date'   },
        { name: 'absenceType',     label: 'Type d\'absence',            value: 'Arrêt maladie professionnelle', type: 'string' },
        { name: 'nbDays',          label: 'Nombre de jours',            value: 52,            type: 'int'    },
        { name: 'employer',        label: 'Employeur',                  value: 'Entreprise Industrielle SA', type: 'string' }
      ]
    },
    {
      id: UUID().toString(),
      individualRecordId: IND,
      title: 'Invalidité catégorie 2',
      type: 'SOCIAL',
      category: 'Invalidité',
      traceability: traceability,
      fields: [
        { name: 'invalidityStatus', label: 'Catégorie d\'invalidité',   value: 'Catégorie 2 - Invalide ne pouvant exercer une activité professionnelle', type: 'string' },
        { name: 'startDate',        label: 'Date d\'effet',             value: '2023-06-01',  type: 'date'   },
        { name: 'origin',           label: 'Origine',                   value: 'Professionnelle (MP)', type: 'string' },
        { name: 'employerInformed', label: 'Employeur informé',         value: true,          type: 'bool'   },
        { name: 'pension',          label: 'Pension mensuelle (€)',      value: 820,           type: 'int'    }
      ]
    },
    {
      id: UUID().toString(),
      individualRecordId: IND,
      title: 'Situation économique 2024',
      type: 'SOCIAL',
      category: 'Situation économique',
      traceability: traceability,
      fields: [
        { name: 'referenceYear',   label: 'Année de référence',         value: 2024,          type: 'int'    },
        { name: 'salary',          label: 'Salaire mensuel net (€)',     value: 1850,          type: 'int'    },
        { name: 'familyAllowances', label: 'Allocations familiales (€)', value: 180,          type: 'int'    },
        { name: 'disability',      label: 'AAH / pension invalidité (€)', value: 820,         type: 'int'    },
        { name: 'familyQuotient',  label: 'Quotient familial',           value: 2.5,          type: 'float'  },
        { name: 'personsInCharge', label: 'Personnes à charge',          value: 2,            type: 'int'    }
      ]
    }
  ];

  if (db.recovery_datas.findOne({_id: RECOVERY_ID})) {
    // Append items to existing
    var result = db.recovery_datas.updateOne(
      {_id: RECOVERY_ID},
      {
        \$push: { items: { \$each: items } },
        \$set:  { updatedAt: NOW },
        \$inc:  { version: 1 }
      }
    );
    print('Items ajoutés à RecoveryData existant : ' + result.modifiedCount);
  } else {
    // Create new
    db.recovery_datas.insertOne({
      _id: RECOVERY_ID,
      id: RECOVERY_ID,
      individualRecordId: IND,
      items: items,
      version: 1,
      createdAt: NOW,
      updatedAt: NOW
    });
    print('RecoveryData créé avec ' + items.length + ' items.');
  }
" 2>/dev/null | tail -3
```

## Étape 6 — Vérification

```bash
echo ""
echo "=== Vérification ==="

COUNT_WRD=$(docker exec workspace-mongodb-1 mongosh --quiet --eval "
  db = db.getSiblingDB('$IR_DB');
  print(db['social.workRelatedDiseases'].countDocuments({individualRecordId: '$IND'}));
" 2>/dev/null | tail -1 | tr -d '\r')

COUNT_RD=$(docker exec workspace-mongodb-1 mongosh --quiet --eval "
  db = db.getSiblingDB('$IR_DB');
  var rd = db.recovery_datas.findOne({individualRecordId: '$IND'});
  print(rd ? rd.items.length : 0);
" 2>/dev/null | tail -1 | tr -d '\r')

echo "  Maladies professionnelles en Social : $COUNT_WRD"
echo "  Items RecoveryData                  : $COUNT_RD"
echo ""
echo "=== Terminé — Individu : $IND ==="
```