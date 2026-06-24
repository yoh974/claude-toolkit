---
name: seed-legacy-occupational-disease
description: "Publie des maladies professionnelles depuis le Connector vers IR via Kafka, avec les valeurs françaises du legacy (recognitionStatus: oui/non/en attente, issuanceContext: interne/externe). Par défaut: dernier individu créé."
argument-hint: "[individualRecordId?]"
allowed-tools: ["Bash"]
---

# seed-legacy-occupational-disease — Seed MP via Kafka Connector

**Syntaxe :** `/seed-legacy-occupational-disease [individualRecordId?]`

Publie 3 maladies professionnelles sur le topic Connector avec les valeurs françaises telles qu'envoyées par le legacy, pour reproduire le comportement réel du Connector.

**Arguments :** "$ARGUMENTS"

## Étape 1 — Résoudre l'individu

```bash
ARGS="$ARGUMENTS"
IND_ARG=$(echo "$ARGS" | awk '{print $1}')
IR_DB="ir-module"
NOW=$(date -u +"%Y-%m-%dT%H:%M:%S.000Z")
CONNECTOR_TOPIC="services/legacy-connector-service.v1alpha.message.OccupationalDisease"
CONTENT_TYPE="services/legacy-connector-service.v1alpha.message.OccupationalDiseaseLegacyImported"

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

## Étape 2 — Publier les 3 maladies professionnelles sur Kafka

```bash
publish_mp() {
  local STATUS="$1" CONTEXT="$2"
  local WRD_ID=$(cat /proc/sys/kernel/random/uuid)
  local EV_ID=$(cat /proc/sys/kernel/random/uuid)
  printf '{"id":"%s","content_type":"%s","created_at":"%s","created_by":"connector","metadata":{},"payload":{"id":"%s","individualRecordId":"%s","occupationalDisease":{"code":"MP057","label":"Affections périarticulaires - Tableau 57"},"observationDate":"2023-09-15T00:00:00.000Z","declarationDate":"2023-10-01T00:00:00.000Z","declarationNumber":2023001,"issuanceContext":"%s","certificateIssuedByDoctor":{"freeText":"Dr. Dupont"},"relapse":false,"consolidation":{"isConsolidated":true,"consolidationDate":"2024-03-15T00:00:00.000Z"},"recovery":{"isRecovered":false},"recognitionStatus":"%s","significant":true}}\n' \
    "$EV_ID" "$CONTENT_TYPE" "$NOW" "$WRD_ID" "$IND" "$CONTEXT" "$STATUS" | \
    docker exec -i workspace-kafka-1 \
      /opt/kafka/bin/kafka-console-producer.sh \
      --bootstrap-server localhost:9092 \
      --topic "$CONNECTOR_TOPIC" 2>/dev/null
  echo "  Publié : recognitionStatus='$STATUS' | issuanceContext='$CONTEXT' | id=$WRD_ID"
}

echo "=== Publication Kafka → $CONNECTOR_TOPIC ==="
publish_mp "oui"        "interne"
publish_mp "non"        "externe"
publish_mp "en attente" "interne"
```

## Étape 3 — Vérification en base après traitement IR

```bash
sleep 2
COUNT=$(docker exec workspace-mongodb-1 mongosh --quiet --eval "
  db = db.getSiblingDB('$IR_DB');
  print(db['social.workRelatedDiseases'].countDocuments({individualRecordId: '$IND'}));
" 2>/dev/null | tail -1 | tr -d '\r')

echo ""
echo "=== Maladies professionnelles en base : $COUNT ==="
echo "=== Terminé — Individu : $IND ==="
```