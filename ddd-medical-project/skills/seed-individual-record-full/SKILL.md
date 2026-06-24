---
name: seed-individual-record-full
description: "Seed complet d'un individu IR via HTTP uniquement (aucun MongoDB direct, aucun Kafka). Couvre : identité (contact), travail (company, job-situation, contract, work-position, exposure-situation, professional-activity, affiliation), santé clinique (biométrie, pathologie, symptôme, examen, traitement, vaccin), santé mentale, modes de vie (alcool, tabac, sommeil, sport, alimentation, drogues), social (life-situation, absence, accident, incapacité, invalidité, maladie pro, parcours, économique). 1-2 entrées par entité. Par défaut : dernier individu créé."
argument-hint: "[individualRecordId?]"
allowed-tools: ["Bash"]
---

# seed-ir-all — Seed complet IR (HTTP only)

**Syntaxe :** `/seed-ir-all [individualRecordId?]`

**Arguments :** "$ARGUMENTS"

## Étape 1 — Résoudre l'individu

```bash
BASE_URL="http://ir.api.localhost/api/v1"
IR_DB="ir-module"
ARGS="$ARGUMENTS"
IND_ARG=$(echo "$ARGS" | awk '{print $1}')

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
echo "=== Individu : $IND ==="
```

## Étape 2 — Pré-générer les IDs et résoudre les codes référentiel

```bash
COMPANY_ID=$(cat /proc/sys/kernel/random/uuid)
JOB_SITUATION_ID=$(cat /proc/sys/kernel/random/uuid)
EXPOSURE_SITUATION_ID=$(cat /proc/sys/kernel/random/uuid)
TODAY=$(date -u +"%Y-%m-%dT00:00:00.000Z")

echo "  COMPANY_ID           : $COMPANY_ID"
echo "  JOB_SITUATION_ID     : $JOB_SITUATION_ID"
echo "  EXPOSURE_SITUATION_ID: $EXPOSURE_SITUATION_ID"

# Helper — résout un item référentiel par regex sur code ou name, fallback sur le premier item
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

PATH_CODE=$(get_ref "M54|M79|DORSALGIE|LOMBAG|PATHOLOG")
SYMP_CODE=$(get_ref "DOULEUR|FATIG|CEPHA|SYMPTOM")
TREAT_CODE=$(get_ref "ANALG|PARAC|MEDIC|TRAITEMENT")
VACC_CODE=$(get_ref "VACCIN|GRIPPE|DTP|COVID")
EXAM_CODE=$(get_ref "RADIO|AUDIO|ECG|BILAN|EXAMEN")
MP_CODE=$(get_ref "MP0|TMS|BRUIT|VIBRAT|MALADIE.PRO")
INVAL_STATUS=$(get_ref "INVALIDI|CAT2|CATEG")

[ -z "$PATH_CODE" ]   && PATH_CODE=$(get_ref ".")
[ -z "$SYMP_CODE" ]   && SYMP_CODE=$(get_ref ".")
[ -z "$TREAT_CODE" ]  && TREAT_CODE=$(get_ref ".")
[ -z "$VACC_CODE" ]   && VACC_CODE=$(get_ref ".")
[ -z "$EXAM_CODE" ]   && EXAM_CODE=$(get_ref ".")
[ -z "$MP_CODE" ]     && MP_CODE=$(get_ref ".")
[ -z "$INVAL_STATUS" ] && INVAL_STATUS=$(get_ref ".")

echo "  pathologyCode : $PATH_CODE"
echo "  symptomCode   : $SYMP_CODE"
echo "  treatmentCode : $TREAT_CODE"
echo "  vaccineCode   : $VACC_CODE"
echo "  examCode      : $EXAM_CODE"
echo "  mpCode        : $MP_CODE"
echo "  invalStatus   : $INVAL_STATUS"
```

## Étape 3 — Identité

```bash
echo ""
echo "=== [1/7] IDENTITÉ ==="

echo "--- identity/information (PUT) ---"
curl -s -o /dev/null -w "  PUT identity/information => HTTP %{http_code}\n" \
  -X PUT "$BASE_URL/individual-records/$IND/identity/information" \
  -H "Content-Type: application/json" \
  -d '{
    "birthName": "DUPONT",
    "maritalName": "DUPONT",
    "usedName": "DUPONT",
    "firstNames": "Jean Marie",
    "usedFirstName": "Jean",
    "birthSex": "M",
    "birthDate": "1980-05-15",
    "civility": "M"
  }'

echo "--- identity/contact/phones (POST) ---"
curl -s -o /dev/null -w "  POST phones => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/individual-records/$IND/identity/contact/phones" \
  -H "Content-Type: application/json" \
  -d '{
    "phoneNumber": "+33612345678",
    "phoneType": "MOBILE",
    "usage": "PERSONAL",
    "isMain": true
  }'

echo "--- identity/contact/emails (POST) ---"
curl -s -o /dev/null -w "  POST emails => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/individual-records/$IND/identity/contact/emails" \
  -H "Content-Type: application/json" \
  -d '{
    "emailAddress": "jean.dupont@example.com",
    "usage": "PERSONAL",
    "isMain": true
  }'
```

## Étape 4 — Travail

```bash
echo ""
echo "=== [2/7] TRAVAIL ==="

echo "--- companies (POST) ---"
curl -s -o /dev/null -w "  POST companies => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/companies" \
  -H "Content-Type: application/json" \
  -d "{
    \"id\": \"$COMPANY_ID\",
    \"establishmentNumber\": \"ETAB-SEED-001\",
    \"commercialName\": \"Acme Corp Seed\",
    \"establishmentType\": \"PME\",
    \"city\": \"Paris\",
    \"postalCode\": \"75001\"
  }"

echo "--- work/job-situations (POST) ---"
curl -s -o /dev/null -w "  POST job-situations => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/work/job-situations" \
  -H "Content-Type: application/json" \
  -d "{
    \"id\": \"$JOB_SITUATION_ID\",
    \"situationType\": \"EMPLOYEE\",
    \"individualRecordId\": \"$IND\",
    \"startDate\": \"2020-01-06\",
    \"employerId\": \"$COMPANY_ID\",
    \"contracts\": [{
      \"type\": \"CONT18\",
      \"startDate\": \"2020-01-06\",
      \"numberOfHours\": 35,
      \"workPositions\": [{
        \"pcsCode\": \"382A\",
        \"title\": \"Développeur logiciel\",
        \"startDate\": \"2020-01-06\"
      }]
    }]
  }"

echo "--- work/job-situations (2e work-position) ---"
CONTRACT_ID=$(curl -s "$BASE_URL/work/job-situations/$JOB_SITUATION_ID" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); c=d.get('contracts',[]); print(c[0]['id'] if c else '')" 2>/dev/null | tr -d '\r')
if [ -n "$CONTRACT_ID" ]; then
  curl -s -o /dev/null -w "  POST work-positions => HTTP %{http_code}\n" \
    -X POST "$BASE_URL/work/job-situations/$JOB_SITUATION_ID/contracts/$CONTRACT_ID/work-positions" \
    -H "Content-Type: application/json" \
    -d "{
      \"pcsCode\": \"211A\",
      \"title\": \"Chef de projet\",
      \"startDate\": \"2022-03-01\"
    }"
else
  echo "  SKIP 2e work-position : contract ID non récupéré"
fi

echo "--- exposures/situations (POST) ---"
curl -s -o /dev/null -w "  POST exposure-situation => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/exposures/situations" \
  -H "Content-Type: application/json" \
  -d "{
    \"id\": \"$EXPOSURE_SITUATION_ID\",
    \"individualRecordId\": \"$IND\",
    \"jobSituationId\": \"$JOB_SITUATION_ID\",
    \"title\": \"Exposition bruit poste assemblage\",
    \"sourceType\": \"POSTE_DE_TRAVAIL\",
    \"sourceValue\": \"Poste assemblage\",
    \"startDate\": {\"year\": \"2020\"},
    \"exposures\": [{
      \"danger\": \"Bruit\",
      \"levelOfExposure\": \"MOYEN\",
      \"origin\": \"PROFESSIONNEL\",
      \"startDate\": {\"year\": \"2020\"}
    }]
  }"

echo "--- professional-activities (POST) ---"
curl -s -o /dev/null -w "  POST professional-activities => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/professional-activities" \
  -H "Content-Type: application/json" \
  -d "{
    \"individualRecordId\": \"$IND\",
    \"startDate\": {\"year\": \"2020\"},
    \"status\": \"ACTIVE\",
    \"description\": \"Développeur logiciel senior\"
  }"

echo "--- affiliations (POST) ---"
curl -s -o /dev/null -w "  POST affiliations => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/affiliations" \
  -H "Content-Type: application/json" \
  -d "{
    \"individualRecordId\": \"$IND\",
    \"jobSituationId\": \"$JOB_SITUATION_ID\",
    \"doctorName\": \"Dr. Martin\",
    \"visitCenterName\": \"Centre Médical Paris Nord\"
  }"
```

## Étape 5 — Santé clinique

```bash
echo ""
echo "=== [3/7] SANTÉ CLINIQUE ==="

echo "--- biometrics ---"
curl -s -o /dev/null -w "  POST biometrics => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/individual-records/$IND/biometrics" \
  -H "Content-Type: application/json" \
  -d '{"size":175,"weight":78,"abdominalPerimeter":88,"heartRate":72,"bloodGroup":"A+","laterality":"RIGHT","systolicBloodPressure":120,"diastolicBloodPressure":80}'

echo "--- pathologies (code: $PATH_CODE) ---"
curl -s -o /dev/null -w "  POST pathologies => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/individual-records/$IND/pathologies" \
  -H "Content-Type: application/json" \
  -d "{\"pathologyCode\":\"$PATH_CODE\",\"startDate\":{\"year\":\"2022\",\"month\":\"3\"},\"medicalHistory\":false,\"workRelatedDisease\":false,\"systems\":[]}"

echo "--- symptoms (code: $SYMP_CODE) ---"
curl -s -o /dev/null -w "  POST symptoms => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/individual-records/$IND/symptoms" \
  -H "Content-Type: application/json" \
  -d "{\"symptomCode\":\"$SYMP_CODE\",\"startDate\":{\"year\":\"2024\",\"month\":\"1\"},\"medicalHistory\":false,\"workRelatedDisease\":false,\"systemsConcerned\":[]}"

echo "--- exams (code: $EXAM_CODE) ---"
curl -s -o /dev/null -w "  POST exams => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/individual-records/$IND/exams" \
  -H "Content-Type: application/json" \
  -d "{
    \"examCode\": \"$EXAM_CODE\",
    \"status\": \"COMPLETED\",
    \"prescriptionFrame\": \"INTERNAL\",
    \"prescriptionDate\": \"$TODAY\",
    \"previsionDate\": \"$TODAY\",
    \"realizationDate\": {\"year\": \"2024\", \"month\": \"3\"},
    \"realizationLocation\": \"INTERNAL\",
    \"systems\": []
  }"

echo "--- treatments (code: $TREAT_CODE) ---"
curl -s -o /dev/null -w "  POST treatments => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/individual-records/$IND/treatments" \
  -H "Content-Type: application/json" \
  -d "{\"treatmentCode\":\"$TREAT_CODE\",\"startDate\":{\"year\":\"2023\",\"month\":\"6\"},\"medicalHistory\":false,\"systemsConcerned\":[]}"

echo "--- vaccins (code: $VACC_CODE) ---"
curl -s -o /dev/null -w "  POST vaccins => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/individual-records/$IND/vaccins" \
  -H "Content-Type: application/json" \
  -d "{\"vaccineCode\":\"$VACC_CODE\",\"vaccineStatus\":\"UP_TO_DATE\",\"systems\":[]}"
```

## Étape 6 — Santé mentale

```bash
echo ""
echo "=== [4/7] SANTÉ MENTALE ==="

curl -s -o /dev/null -w "  POST mental-health => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/individual-records/$IND/mental-health" \
  -H "Content-Type: application/json" \
  -d '{"generalMentalHealth":7,"workFeeling":6,"workStress":5,"wellBeing":7,"psychosocialRisks":4,"startDate":{"year":"2024","month":"1"}}'
```

## Étape 7 — Modes de vie

```bash
echo ""
echo "=== [5/7] MODES DE VIE ==="

echo "--- alcohols ---"
curl -s -o /dev/null -w "  POST alcohols => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/individual-records/$IND/alcohols" \
  -H "Content-Type: application/json" \
  -d '{"status":"Consommation occasionnelle","quantity":2,"startDate":{"year":"2024"}}'

echo "--- tobaccos ---"
curl -s -o /dev/null -w "  POST tobaccos => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/individual-records/$IND/tobaccos" \
  -H "Content-Type: application/json" \
  -d '{"smokerStatus":"FORMER_SMOKER","passiveSmokingExposure":"LOW","dailyConsumption":{"value":10,"unit":"CIGARETS_PER_DAY"},"thresholdConsumption":"MODERATE","startDate":{"year":"2015"},"endDate":{"year":"2022"}}'

echo "--- sleeps ---"
curl -s -o /dev/null -w "  POST sleeps => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/individual-records/$IND/sleeps" \
  -H "Content-Type: application/json" \
  -d '{"quantity":7,"quality":"CORRECT","threshold":"RESTORATIVE","startDate":{"year":"2024"}}'

echo "--- sports ---"
curl -s -o /dev/null -w "  POST sports => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/individual-records/$IND/sports" \
  -H "Content-Type: application/json" \
  -d '{"activityType":"Natation","practiceFrequency":3,"practiceThreshold":"MODERATE","startDate":{"year":"2023"}}'

echo "--- foods ---"
curl -s -o /dev/null -w "  POST foods => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/individual-records/$IND/foods" \
  -H "Content-Type: application/json" \
  -d '{"hydrationLevel":1.5,"hydrationThreshold":"EQUILIBREE","feedingFrequency":3,"foodQuality":"RECOMANDEE","startDate":{"year":"2024"}}'

echo "--- drugs ---"
curl -s -o /dev/null -w "  POST drugs => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/individual-records/$IND/drugs" \
  -H "Content-Type: application/json" \
  -d '{"drugType":"CANNABIS","frequencyOfUse":{"value":1,"unit":"PER_WEEK"},"threshold":"CASUAL","impactOnWork":"NONE","startDate":{"year":"2022"},"endDate":{"year":"2023"}}'
```

## Étape 8 — Social

```bash
echo ""
echo "=== [6/7] SOCIAL ==="

echo "--- life-situations ---"
curl -s -o /dev/null -w "  POST life-situations => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/individual-records/$IND/life-situations" \
  -H "Content-Type: application/json" \
  -d '{"startDate":{"year":"2023"},"childrenLivingWith":2,"childrenBirthYears":[2015,2018],"personsLivingWith":3}'

echo "--- absences ---"
curl -s -o /dev/null -w "  POST absences => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/individual-records/$IND/absences" \
  -H "Content-Type: application/json" \
  -d '{"typeId":"ARRET_MALADIE","startDate":{"year":"2024","month":"1"},"endDate":{"year":"2024","month":"3"},"employers":[],"nbDays":60}'

echo "--- work-accidents ---"
curl -s -o /dev/null -w "  POST work-accidents => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/individual-records/$IND/work-accidents" \
  -H "Content-Type: application/json" \
  -d '{"accidentType":"Chute de plain-pied","startDate":{"year":"2023","month":"11"},"endDate":{"year":"2023","month":"12"},"workLinkRecognition":true,"employersConcerned":[],"otherEmployer":"Acme Corp Seed"}'

echo "--- work-incapacities ---"
curl -s -o /dev/null -w "  POST work-incapacities => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/individual-records/$IND/work-incapacities" \
  -H "Content-Type: application/json" \
  -d '{"startDate":{"year":"2024","month":"1"},"endDate":{"year":"2024","month":"6"},"disabilityType":"TEMPORARY","disabilityDetails":"Lombalgie chronique invalidante","origin":"WORK_ACCIDENT"}'

if [ -n "$INVAL_STATUS" ]; then
  echo "--- work-invalidities (invalStatus: $INVAL_STATUS) ---"
  curl -s -o /dev/null -w "  POST work-invalidities => HTTP %{http_code}\n" \
    -X POST "$BASE_URL/individual-records/$IND/work-invalidities" \
    -H "Content-Type: application/json" \
    -d "{\"invalidityStatus\":\"$INVAL_STATUS\",\"startDate\":{\"year\":\"2022\",\"month\":\"6\"},\"origin\":\"NON_PROFESSIONAL\",\"employerIsInformedOfInvalidityStatus\":true}"
else
  echo "  SKIP work-invalidities : aucun item référentiel trouvé"
fi

echo "--- economic-situations ---"
curl -s -o /dev/null -w "  POST economic-situations => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/individual-records/$IND/economic-situations" \
  -H "Content-Type: application/json" \
  -d '{"startDate":{"year":"2024","month":"1"},"householdDisposableIncomes":[{"incomeName":"Salaire","amount":2200},{"incomeName":"Allocations familiales","amount":180}],"familyQuotient":2.5}'

echo "--- support-journeys ---"
curl -s -o /dev/null -w "  POST support-journeys => HTTP %{http_code}\n" \
  -X POST "$BASE_URL/individual-records/$IND/support-journeys" \
  -H "Content-Type: application/json" \
  -d '{"objective":"Maintien dans l'\''emploi","startDate":{"year":"2024","month":"1"},"prescriptionFramework":"INTERNAL","hasProfessionalOrigin":true,"prescriber":"Dr. Martin"}'

if [ -n "$MP_CODE" ]; then
  echo "--- work-related-diseases (mpCode: $MP_CODE) ---"
  curl -s -o /dev/null -w "  POST work-related-diseases => HTTP %{http_code}\n" \
    -X POST "$BASE_URL/individual-records/$IND/work-related-diseases" \
    -H "Content-Type: application/json" \
    -d "{
      \"workRelatedDisease\": \"$MP_CODE\",
      \"companies\": [{\"text\": \"Acme Corp Seed\"}],
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
else
  echo "  SKIP work-related-diseases : aucun item référentiel trouvé"
fi
```

## Étape 9 — Vérification rapide

```bash
echo ""
echo "=== [7/7] VÉRIFICATION ==="

COUNT=$(docker exec workspace-mongodb-1 mongosh --quiet --eval "
  var db2 = db.getSiblingDB('$IR_DB');
  var t = new Date(Date.now() - 5*60*1000);
  print(db2.events.countDocuments({created_at:{\$gte:t}}));
" 2>/dev/null | tail -1 | tr -d '\r')

echo "  Événements créés (5 dernières min) : $COUNT"
echo ""
echo "=== Terminé ==="
echo "  Individu         : $IND"
echo "  Company          : $COMPANY_ID"
echo "  JobSituation     : $JOB_SITUATION_ID"
echo "  ExposureSituation: $EXPOSURE_SITUATION_ID"
```