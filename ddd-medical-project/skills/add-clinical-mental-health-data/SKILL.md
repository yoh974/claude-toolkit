---
name: add-clinical-mental-health-data
description: "Ajoute des données de santé clinique (biométrie, pathologies, symptômes, traitements, vaccins, examens) et santé mentale à un individu IR. Vérifie les agrégats en base. Par défaut: dernier individu créé."
argument-hint: "[entity] [individualRecordId?]  — entity: biometric|pathology|symptom|treatment|vaccine|mental-health|clinical-health|all (défaut)"
allowed-tools: ["Bash"]
---

# add-ir-health-clinical-and-mental — Seed clinical & mental health data

**Syntaxe :** `/add-ir-health-clinical-and-mental [entity] [individualRecordId?]`

Entités : `biometric`, `pathology`, `symptom`, `treatment`, `vaccine`, `mental-health`, `clinical-health` (biometric+pathology), `all` (défaut)

**Arguments :** "$ARGUMENTS"

## Étape 1 — Parser et résoudre l'individu

```bash
ARGS="$ARGUMENTS"
ENTITY=$(echo "$ARGS" | awk '{print $1}')
IND_ARG=$(echo "$ARGS" | awk '{print $2}')
ENTITY="${ENTITY:-all}"
BASE_URL="http://ir.api.localhost/api/v1/individual-records"
IR_DB="ir-module"
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

## Étape 2 — Fonction ensure_aggregate (vérifie/crée l'agrégat en MongoDB)

```bash
ensure_aggregate() {
  local AGG_TYPE="$1"
  local COLLECTION="$2"
  local EXISTS=$(docker exec workspace-mongodb-1 mongosh --quiet --eval "
    db = db.getSiblingDB('$IR_DB');
    var s = db['$COLLECTION'].findOne({individualRecordId: '$IND'}, {_id:1});
    print(s ? 'yes' : 'no');
  " 2>/dev/null | tail -1 | tr -d '\r')

  if [ "$EXISTS" = "yes" ]; then
    echo "  Agrégat $AGG_TYPE présent."
    return 0
  fi

  echo "  Agrégat $AGG_TYPE absent — création en MongoDB..."
  local AGG_ID=$(cat /proc/sys/kernel/random/uuid)
  local EVENT_ID=$(cat /proc/sys/kernel/random/uuid)
  docker exec workspace-mongodb-1 mongosh --quiet --eval "
    db = db.getSiblingDB('$IR_DB');
    var NOW = new Date('$NOW');
    db.events.insertOne({
      _id:'$EVENT_ID',content_type:'${AGG_TYPE}Created',created_at:NOW,created_by:'system',
      aggregate_id:'$AGG_ID',aggregate_type:'$AGG_TYPE',version:1,
      metadata:{'x-request-id':null,'saga-context':null,'aggregate-id':'$AGG_ID','aggregate-type':'$AGG_TYPE','tenant-id':null,token:null,locale:null},
      payload:{id:'$AGG_ID',individualRecordId:'$IND',createdAt:'$NOW',updatedAt:'$NOW'}
    });
    db['$COLLECTION'].insertOne({_id:'$AGG_ID',id:'$AGG_ID',individualRecordId:'$IND',version:1,createdAt:NOW,updatedAt:NOW});
    print('  => $AGG_TYPE créé : $AGG_ID');
  " 2>/dev/null | tail -2
}
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

## Étape 4 — Fonctions d'ajout

### Clinical Health

```bash
add_biometric() {
  ensure_aggregate "ClinicalHealth" "health.clinical_health"
  echo "--- biometric ---"
  curl -s -X POST "$BASE_URL/$IND/biometrics" -H "Content-Type: application/json" \
    -d '{"size":175,"weight":78,"abdominalPerimeter":88,"heartRate":72,"bloodGroup":"A+","laterality":"RIGHT","systolicBloodPressure":120,"diastolicBloodPressure":80}' \
    | python3 -c "import sys,json;d=json.load(sys.stdin);print('  =>',d)" 2>/dev/null
}

add_pathology() {
  ensure_aggregate "ClinicalHealth" "health.clinical_health"
  local CODE=$(get_ref_item "^M54|LOMBALG|DORSALG")
  if [ -z "$CODE" ]; then CODE=$(get_ref_item "."); fi
  echo "--- pathology (pathologyCode: $CODE) ---"
  curl -s -X POST "$BASE_URL/$IND/pathologies" -H "Content-Type: application/json" \
    -d "{\"pathologyCode\":\"$CODE\",\"startDate\":{\"year\":\"2023\",\"month\":\"01\"},\"medicalHistory\":false,\"workRelatedDisease\":false,\"systems\":[]}" \
    | python3 -c "import sys,json;d=json.load(sys.stdin);print('  =>',d)" 2>/dev/null
}

add_symptom() {
  ensure_aggregate "ClinicalHealth" "health.clinical_health"
  local CODE=$(get_ref_item "DOULEUR|FATIG|SYMPTOM")
  if [ -z "$CODE" ]; then CODE=$(get_ref_item "."); fi
  echo "--- symptom (symptomCode: $CODE) ---"
  curl -s -X POST "$BASE_URL/$IND/symptoms" -H "Content-Type: application/json" \
    -d "{\"symptomCode\":\"$CODE\",\"medicalHistory\":false,\"workRelatedDisease\":false,\"startDate\":{\"year\":\"2024\",\"month\":\"01\"},\"systemsConcerned\":[]}" \
    | python3 -c "import sys,json;d=json.load(sys.stdin);print('  =>',d)" 2>/dev/null
}

add_treatment() {
  ensure_aggregate "ClinicalHealth" "health.clinical_health"
  local CODE=$(get_ref_item "TRAITEMENT|MEDIC|ANALG")
  if [ -z "$CODE" ]; then CODE=$(get_ref_item "."); fi
  echo "--- treatment (treatmentCode: $CODE) ---"
  curl -s -X POST "$BASE_URL/$IND/treatments" -H "Content-Type: application/json" \
    -d "{\"treatmentCode\":\"$CODE\",\"medicalHistory\":false,\"startDate\":{\"year\":\"2023\",\"month\":\"06\"},\"systemsConcerned\":[]}" \
    | python3 -c "import sys,json;d=json.load(sys.stdin);print('  =>',d)" 2>/dev/null
}

add_vaccine() {
  ensure_aggregate "ClinicalHealth" "health.clinical_health"
  local CODE=$(get_ref_item "VACCIN|GRIPPE|COVID|DTP")
  if [ -z "$CODE" ]; then CODE=$(get_ref_item "."); fi
  echo "--- vaccine (vaccineCode: $CODE) ---"
  curl -s -X POST "$BASE_URL/$IND/vaccins" -H "Content-Type: application/json" \
    -d "{\"vaccineCode\":\"$CODE\",\"systems\":[]}" \
    | python3 -c "import sys,json;d=json.load(sys.stdin);print('  =>',d)" 2>/dev/null
}
```

### Mental Health

```bash
add_mental_health() {
  ensure_aggregate "MentalHealth" "health.mental_health"
  echo "--- mental-health ---"
  curl -s -X POST "$BASE_URL/$IND/mental-health" -H "Content-Type: application/json" \
    -d '{"generalMentalHealth":7,"workFeeling":6,"workStress":5,"wellBeing":7,"psychosocialRisks":4,"startDate":{"year":"2024","month":"01"}}' \
    | python3 -c "import sys,json;d=json.load(sys.stdin);print('  =>',d)" 2>/dev/null
}
```

## Étape 5 — Dispatcher

```bash
case "$ENTITY" in
  biometric)     add_biometric ;;
  pathology)     add_pathology ;;
  symptom)       add_symptom ;;
  treatment)     add_treatment ;;
  vaccine)       add_vaccine ;;
  mental-health) add_mental_health ;;
  clinical-health)
    add_biometric
    add_pathology
    ;;
  all)
    add_biometric
    add_pathology
    add_symptom
    add_treatment
    add_vaccine
    add_mental_health
    ;;
  *)
    echo "Entité inconnue: $ENTITY. Valeurs: biometric, pathology, symptom, treatment, vaccine, mental-health, clinical-health, all"
    exit 1 ;;
esac

echo ""
echo "=== Terminé — Individu : $IND ==="
```
