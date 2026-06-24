---
name: add-social-data
description: "Ajoute des données sociales (parcours d'accompagnement, vie, accidents, incapacités, invalidités, absences, maladies professionnelles, économique) à un individu IR via HTTP. Par défaut: dernier individu créé."
argument-hint: "[entity] [individualRecordId?]  — entity: support-journey|life-situation|work-accident|work-incapacity|work-invalidity|work-absence|economic-situation|work-related-disease|all (défaut)"
allowed-tools: ["Bash"]
---

# add-ir-social — Seed social data via HTTP

**Syntaxe :** `/add-ir-social [entity] [individualRecordId?]`

Entités : `support-journey`, `life-situation`, `work-accident`, `work-incapacity`, `work-invalidity`, `work-absence`, `economic-situation`, `work-related-disease`, `all` (défaut)

**Arguments :** "$ARGUMENTS"

## Étape 1 — Parser et résoudre l'individu

```bash
ARGS="$ARGUMENTS"
ENTITY=$(echo "$ARGS" | awk '{print $1}')
IND_ARG=$(echo "$ARGS" | awk '{print $2}')
ENTITY="${ENTITY:-all}"
BASE_URL="http://ir.api.localhost/api/v1/individual-records"
IR_DB="ir-module"

if [ -z "$IND_ARG" ]; then
  IND=$(docker exec workspace-mongodb-1 mongosh --quiet --eval "
    db = db.getSiblingDB('$IR_DB');
    var ev = db.events.findOne(
      {aggregate_type:'Identity'},
      {'payload.individualRecordId':1},
      {sort:{created_at:-1}}
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

## Étape 2 — Résoudre les codes référentiel nécessaires

```bash
get_ref() {
  local PATTERN="$1"
  docker exec workspace-mongodb-1 mongosh --quiet --eval "
    var db2 = db.getSiblingDB('$IR_DB');
    var it = db2['referentiel.items'].findOne({code:{\$regex:'$PATTERN',\$options:'i'}},{_id:1})
          || db2['referentiel.items'].findOne({name:{\$regex:'$PATTERN',\$options:'i'}},{_id:1})
          || db2['referentiel.items'].findOne({},{_id:1});
    print(it ? it._id : '');
  " 2>/dev/null | tail -1 | tr -d '\r'
}

MP_CODE=$(get_ref "MP0|TMS|BRUIT|VIBRAT|MALADIE.PRO")
INVAL_STATUS=$(get_ref "INVALIDI|CAT2|CATEG")
[ -z "$MP_CODE" ]     && MP_CODE=$(get_ref ".")
[ -z "$INVAL_STATUS" ] && INVAL_STATUS=$(get_ref ".")

echo "  mpCode      : $MP_CODE"
echo "  invalStatus : $INVAL_STATUS"
```

## Étape 3 — Fonctions d'ajout (HTTP uniquement)

```bash
add_support_journey() {
  echo "--- support-journey ---"
  curl -s -o /dev/null -w "  POST support-journeys => HTTP %{http_code}\n" \
    -X POST "$BASE_URL/$IND/support-journeys" \
    -H "Content-Type: application/json" \
    -d '{
      "objective": "Maintien dans l'\''emploi suite à maladie professionnelle",
      "startDate": {"year": "2024", "month": "1"},
      "prescriptionFramework": "INTERNAL",
      "hasProfessionalOrigin": true,
      "prescriber": "Dr. Martin"
    }'
}

add_life_situation() {
  echo "--- life-situation ---"
  curl -s -o /dev/null -w "  POST life-situations => HTTP %{http_code}\n" \
    -X POST "$BASE_URL/$IND/life-situations" \
    -H "Content-Type: application/json" \
    -d '{
      "startDate": {"year": "2023"},
      "childrenLivingWith": 2,
      "childrenBirthYears": [2015, 2018],
      "personsLivingWith": 3
    }'
}

add_work_accident() {
  echo "--- work-accident ---"
  curl -s -o /dev/null -w "  POST work-accidents => HTTP %{http_code}\n" \
    -X POST "$BASE_URL/$IND/work-accidents" \
    -H "Content-Type: application/json" \
    -d '{
      "accidentType": "Chute de plain-pied",
      "startDate": {"year": "2023", "month": "11"},
      "endDate": {"year": "2023", "month": "12"},
      "workLinkRecognition": true,
      "employersConcerned": [],
      "otherEmployer": "Entreprise Test SAS"
    }'
}

add_work_incapacity() {
  echo "--- work-incapacity ---"
  curl -s -o /dev/null -w "  POST work-incapacities => HTTP %{http_code}\n" \
    -X POST "$BASE_URL/$IND/work-incapacities" \
    -H "Content-Type: application/json" \
    -d '{
      "startDate": {"year": "2024", "month": "1"},
      "endDate": {"year": "2024", "month": "3"},
      "disabilityType": "TEMPORARY",
      "disabilityDetails": "Lombalgie chronique invalidante",
      "origin": "WORK_ACCIDENT"
    }'
}

add_work_invalidity() {
  if [ -z "$INVAL_STATUS" ]; then
    echo "  SKIP work-invalidity : aucun item référentiel pour invalidityStatus"
    return
  fi
  echo "--- work-invalidity (invalStatus: $INVAL_STATUS) ---"
  curl -s -o /dev/null -w "  POST work-invalidities => HTTP %{http_code}\n" \
    -X POST "$BASE_URL/$IND/work-invalidities" \
    -H "Content-Type: application/json" \
    -d "{
      \"invalidityStatus\": \"$INVAL_STATUS\",
      \"startDate\": {\"year\": \"2023\", \"month\": \"6\"},
      \"origin\": \"NON_PROFESSIONAL\",
      \"employerIsInformedOfInvalidityStatus\": true
    }"
}

add_work_absence() {
  echo "--- work-absence ---"
  curl -s -o /dev/null -w "  POST absences => HTTP %{http_code}\n" \
    -X POST "$BASE_URL/$IND/absences" \
    -H "Content-Type: application/json" \
    -d '{
      "typeId": "ARRET_MALADIE",
      "startDate": {"year": "2024", "month": "1"},
      "endDate": {"year": "2024", "month": "2"},
      "employers": [],
      "nbDays": 52
    }'
}

add_economic_situation() {
  echo "--- economic-situation ---"
  curl -s -o /dev/null -w "  POST economic-situations => HTTP %{http_code}\n" \
    -X POST "$BASE_URL/$IND/economic-situations" \
    -H "Content-Type: application/json" \
    -d '{
      "startDate": {"year": "2024", "month": "1"},
      "householdDisposableIncomes": [
        {"incomeName": "Salaire", "amount": 2200},
        {"incomeName": "Allocations familiales", "amount": 180}
      ],
      "familyQuotient": 2.5
    }'
}

add_work_related_disease() {
  if [ -z "$MP_CODE" ]; then
    echo "  SKIP work-related-disease : aucun item référentiel trouvé"
    return
  fi
  echo "--- work-related-disease (mpCode: $MP_CODE) ---"
  curl -s -o /dev/null -w "  POST work-related-diseases => HTTP %{http_code}\n" \
    -X POST "$BASE_URL/$IND/work-related-diseases" \
    -H "Content-Type: application/json" \
    -d "{
      \"workRelatedDisease\": \"$MP_CODE\",
      \"companies\": [{\"text\": \"Entreprise Industrielle SA\"}],
      \"exposureRiskConcerns\": [{\"text\": \"Vibrations mécaniques\"}],
      \"observationDate\": \"2023-09-15\",
      \"declarationDate\": \"2023-10-01\",
      \"declarationNumber\": 2023001,
      \"certificateDelivered\": true,
      \"certificateDeliveryFrame\": \"INTERNAL\",
      \"certificateHealthProfessionalFreeText\": \"Dr. Dupont\",
      \"relapse\": {\"occurred\": false},
      \"consolidation\": {\"occurred\": true, \"date\": \"2024-03-15\"},
      \"healing\": {\"occurred\": false},
      \"recognitionStatus\": \"ACCEPTED\"
    }"
}
```

## Étape 4 — Dispatcher

```bash
case "$ENTITY" in
  support-journey)      add_support_journey ;;
  life-situation)       add_life_situation ;;
  work-accident)        add_work_accident ;;
  work-incapacity)      add_work_incapacity ;;
  work-invalidity)      add_work_invalidity ;;
  work-absence)         add_work_absence ;;
  economic-situation)   add_economic_situation ;;
  work-related-disease) add_work_related_disease ;;
  all)
    add_support_journey
    add_life_situation
    add_work_accident
    add_work_incapacity
    add_work_invalidity
    add_work_absence
    add_economic_situation
    add_work_related_disease
    ;;
  *)
    echo "Entité inconnue: '$ENTITY'"
    echo "Valeurs : support-journey, life-situation, work-accident, work-incapacity, work-invalidity, work-absence, economic-situation, work-related-disease, all"
    exit 1 ;;
esac

echo ""
echo "=== Terminé — Individu : $IND ==="
```