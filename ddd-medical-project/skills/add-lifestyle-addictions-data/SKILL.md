---
name: add-lifestyle-addictions-data
description: "Ajoute des données de modes de vie et addictions (alcool, tabac, sommeil, sport, alimentation, drogues) à un individu IR. Vérifie l'agrégat LifestyleAndAddictions en base. Par défaut: dernier individu créé."
argument-hint: "[entity] [individualRecordId?]  — entity: alcohol|tobacco|sleep|sport|food|drug|all (défaut)"
allowed-tools: ["Bash"]
---

# add-ir-health-lifestyle — Seed lifestyle & addictions data

**Syntaxe :** `/add-ir-health-lifestyle [entity] [individualRecordId?]`

Entités : `alcohol`, `tobacco`, `sleep`, `sport`, `food`, `drug`, `all` (défaut)

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

## Étape 2 — Vérifier/créer l'agrégat LifestyleAndAddictions

```bash
LIFESTYLE_EXISTS=$(docker exec workspace-mongodb-1 mongosh --quiet --eval "
  db = db.getSiblingDB('$IR_DB');
  var s = db['health.lifestyle_and_addictions'].findOne({individualRecordId: '$IND'}, {_id:1});
  print(s ? 'yes' : 'no');
" 2>/dev/null | tail -1 | tr -d '\r')

if [ "$LIFESTYLE_EXISTS" != "yes" ]; then
  echo "Agrégat LifestyleAndAddictions absent — création en MongoDB..."
  AGG_ID=$(cat /proc/sys/kernel/random/uuid)
  EVENT_ID=$(cat /proc/sys/kernel/random/uuid)
  docker exec workspace-mongodb-1 mongosh --quiet --eval "
    db = db.getSiblingDB('$IR_DB');
    var NOW = new Date('$NOW');
    db.events.insertOne({
      _id:'$EVENT_ID',content_type:'LifestyleAndAddictionsCreated',created_at:NOW,created_by:'system',
      aggregate_id:'$AGG_ID',aggregate_type:'LifestyleAndAddictions',version:1,
      metadata:{'x-request-id':null,'saga-context':null,'aggregate-id':'$AGG_ID','aggregate-type':'LifestyleAndAddictions','tenant-id':null,token:null,locale:null},
      payload:{id:'$AGG_ID',individualRecordId:'$IND',lifestyles:[],addictions:[],createdAt:'$NOW',updatedAt:'$NOW'}
    });
    db['health.lifestyle_and_addictions'].insertOne({_id:'$AGG_ID',id:'$AGG_ID',individualRecordId:'$IND',version:1,createdAt:NOW,updatedAt:NOW});
    print('LifestyleAndAddictions créé : $AGG_ID');
  " 2>/dev/null | tail -2
else
  echo "Agrégat LifestyleAndAddictions présent."
fi
```

## Étape 3 — Résoudre un item référentiel (pour sport/activityType)

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

Enums disponibles :
- AlcoholStatus : `"Aucune consommation"`, `"Consommation occasionnelle"`, `"Consommation régulière"`, `"Consommation abusive"`
- SmokerStatus : `ACTIVE_SMOKER`, `FORMER_SMOKER`, `NEVER_SMOKED`
- PassiveSmokingExposure : `LOW`, `MODERATE`, `HIGH`
- ThresholdConsumption : `LOW`, `MODERATE`, `HIGH`
- UnitDailyConsumption : `CIGARETS_PER_DAY`, `PACKS_PER_WEEK`
- SleepQuality : `MEDIOCRE`, `CORRECT`, `RESTORATIVE`
- SleepThreshold : `RISK`, `RESTORATIVE`
- HydrationThreshold : `EQUILIBREE`, `DESEQUILIBREE`
- FoodQuality : `RECOMANDEE` (à vérifier selon les valeurs complètes)
- SportPracticeThreshold : `NONE`, `LOW`, `MODERATE`, `INTENSE`
- DrugTypeEnum : `CANNABIS`, `COCAINE`, `AMPHETAMINE`, `OPIATE`, `BENZODIAZEPINE`, `HALLUCINOGENIC`, `SOLVENTS`
- FrequencyUnits : `PER_WEEK`
- ThresholdEnum : `CASUAL`, `REGULAR`, `DEPENDANCE`
- ImpactOnWorkEnum : `NONE`, `MODERATE`, `SEVERE`

```bash
add_alcohol() {
  echo "--- alcohol ---"
  curl -s -X POST "$BASE_URL/$IND/alcohols" -H "Content-Type: application/json" \
    -d '{"status":"Consommation occasionnelle","quantity":2,"startDate":{"year":"2024"}}' \
    | python3 -c "import sys,json;d=json.load(sys.stdin);print('  =>',d)" 2>/dev/null
}

add_tobacco() {
  echo "--- tobacco ---"
  curl -s -X POST "$BASE_URL/$IND/tobaccos" -H "Content-Type: application/json" \
    -d '{"smokerStatus":"FORMER_SMOKER","passiveSmokingExposure":"LOW","dailyConsumption":{"value":10,"unit":"CIGARETS_PER_DAY"},"thresholdConsumption":"MODERATE","startDate":{"year":"2020"},"endDate":{"year":"2023"}}' \
    | python3 -c "import sys,json;d=json.load(sys.stdin);print('  =>',d)" 2>/dev/null
}

add_sleep() {
  echo "--- sleep ---"
  curl -s -X POST "$BASE_URL/$IND/sleeps" -H "Content-Type: application/json" \
    -d '{"quantity":7,"quality":"CORRECT","threshold":"RESTORATIVE","startDate":{"year":"2024"},"endDate":{"year":"2024","month":"12"}}' \
    | python3 -c "import sys,json;d=json.load(sys.stdin);print('  =>',d)" 2>/dev/null
}

add_sport() {
  local SPORT_ID=$(get_ref_item "SPORT|NATATION|COURSE|VELO|MARCHE")
  if [ -z "$SPORT_ID" ]; then SPORT_ID=$(get_ref_item "."); fi
  echo "--- sport (activityType: $SPORT_ID) ---"
  curl -s -X POST "$BASE_URL/$IND/sports" -H "Content-Type: application/json" \
    -d "{\"activityType\":\"$SPORT_ID\",\"practiceFrequency\":3,\"practiceThreshold\":\"MODERATE\",\"startDate\":{\"year\":\"2023\"},\"endDate\":{\"year\":\"2024\"}}" \
    | python3 -c "import sys,json;d=json.load(sys.stdin);print('  =>',d)" 2>/dev/null
}

add_food() {
  echo "--- food ---"
  curl -s -X POST "$BASE_URL/$IND/foods" -H "Content-Type: application/json" \
    -d '{"hydrationLevel":1.5,"hydrationThreshold":"EQUILIBREE","feedingFrequency":3,"foodQuality":"RECOMANDEE","startDate":{"year":"2024"},"endDate":{"year":"2024","month":"12"}}' \
    | python3 -c "import sys,json;d=json.load(sys.stdin);print('  =>',d)" 2>/dev/null
}

add_drug() {
  echo "--- drug ---"
  curl -s -X POST "$BASE_URL/$IND/drugs" -H "Content-Type: application/json" \
    -d '{"drugType":"CANNABIS","frequencyOfUse":{"value":1,"unit":"PER_WEEK"},"threshold":"CASUAL","impactOnWork":"NONE","startDate":{"year":"2022"},"endDate":{"year":"2023"}}' \
    | python3 -c "import sys,json;d=json.load(sys.stdin);print('  =>',d)" 2>/dev/null
}
```

## Étape 5 — Dispatcher

```bash
case "$ENTITY" in
  alcohol) add_alcohol ;;
  tobacco) add_tobacco ;;
  sleep)   add_sleep ;;
  sport)   add_sport ;;
  food)    add_food ;;
  drug)    add_drug ;;
  all)
    add_alcohol
    add_tobacco
    add_sleep
    add_sport
    add_food
    add_drug
    ;;
  *)
    echo "Entité inconnue: $ENTITY. Valeurs: alcohol, tobacco, sleep, sport, food, drug, all"
    exit 1 ;;
esac

echo ""
echo "=== Terminé — Individu : $IND ==="
```
