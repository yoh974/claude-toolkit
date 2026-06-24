---
name: seed-medical-visit
description: "Crée une visite médicale (MV) via le Connector pour un individu IR. Par défaut: dernier individu IR créé."
argument-hint: "[personId?] [--performer-id <uuid>] [--verbose] [--check]"
allowed-tools: ["Bash"]
---

# seed-mv — Créer une visite médicale

**Syntaxe :** `/seed-mv [personId?] [--performer-id <uuid>] [--verbose] [--check]`

- `personId` : UUID de l'individu IR (optionnel — par défaut: dernier créé)
- `--performer-id` : UUID du professionnel de santé (optionnel)
- `--verbose` : Affiche le payload complet et la réponse serveur
- `--check` : Teste uniquement la connexion au Connector

**Arguments :** "$ARGUMENTS"

## Étape 1 — Parser les arguments

```bash
ARGS="$ARGUMENTS"
PERSON_ID=""
PERFORMER_ID=""
VERBOSE=""
CHECK=""

# Parser les flags
REMAINING=""
while [ -n "$ARGS" ]; do
  TOKEN=$(echo "$ARGS" | awk '{print $1}')
  REST=$(echo "$ARGS" | cut -d' ' -f2-)
  [ "$REST" = "$TOKEN" ] && REST=""

  case "$TOKEN" in
    --verbose) VERBOSE="--verbose" ; ARGS="$REST" ;;
    --check)   CHECK="--check"     ; ARGS="$REST" ;;
    --performer-id)
      PERFORMER_ID=$(echo "$REST" | awk '{print $1}')
      ARGS=$(echo "$REST" | cut -d' ' -f2-)
      [ "$ARGS" = "$PERFORMER_ID" ] && ARGS=""
      ;;
    *) REMAINING="$TOKEN" ; ARGS="$REST" ;;
  esac
done

[ -n "$REMAINING" ] && PERSON_ID="$REMAINING"
```

## Étape 2 — Résoudre le personId si absent

```bash
TOOLS_DIR="$WORKSPACE_DIR/tools"
IR_DB="ir-module"

if [ -z "$CHECK" ] && [ -z "$PERSON_ID" ]; then
  echo "Aucun personId fourni — recherche du dernier individu IR..."
  PERSON_ID=$(docker exec workspace-mongodb-1 mongosh --quiet --eval "
    db = db.getSiblingDB('$IR_DB');
    var ev = db.events.findOne(
      {aggregate_type:'Identity'},
      {'payload.individualRecordId': 1},
      {sort: {created_at: -1}}
    );
    print(ev && ev.payload ? ev.payload.individualRecordId : 'NOT_FOUND');
  " 2>/dev/null | tail -1 | tr -d '\r')

  if [ "$PERSON_ID" = "NOT_FOUND" ] || [ -z "$PERSON_ID" ]; then
    echo "ERREUR: aucun individu IR trouvé en base."
    echo "Créez d'abord un individu avec: npx tsx tools/index.ts ir --total 1"
    exit 1
  fi
  echo "PersonId résolu : $PERSON_ID"
fi
```

## Étape 3 — Construire et exécuter la commande

```bash
CMD="npx tsx index.ts visit"

[ -n "$CHECK" ]        && CMD="$CMD --check"
[ -n "$PERSON_ID" ]    && CMD="$CMD --person-id $PERSON_ID"
[ -n "$PERFORMER_ID" ] && CMD="$CMD --performer-id $PERFORMER_ID"
[ -n "$VERBOSE" ]      && CMD="$CMD --verbose"

echo "Commande : $CMD"
echo ""

OUTPUT=$(cd "$TOOLS_DIR" && $CMD)
echo "$OUTPUT"
```

## Étape 4 — Vérifier la création en base MongoDB

```bash
# Skip si --check
if [ -z "$CHECK" ]; then
  VISIT_ID=$(echo "$OUTPUT" | grep -oP 'visitId: \K[0-9a-f-]{36}')

  if [ -z "$VISIT_ID" ]; then
    echo "⚠️  Impossible d'extraire le visitId depuis la réponse."
  else
    echo ""
    echo "🔍 Vérification MongoDB (medical-visit-db)..."
    FOUND=$(docker exec workspace-mongodb-1 mongosh --quiet --eval "
      db = db.getSiblingDB('medical-visit-db');
      var doc = db['medical-visit-form-content_projection'].findOne({visitId: '$VISIT_ID'}, {visitId:1});
      print(doc ? 'FOUND' : 'NOT_FOUND');
    " 2>/dev/null | tail -1 | tr -d '\r')

    if [ "$FOUND" = "FOUND" ]; then
      echo "✅ Visite $VISIT_ID confirmée en base."
    else
      echo "❌ Visite $VISIT_ID NON trouvée en base — le message Kafka n'a peut-être pas encore été traité."
      echo "   Vérifiez les logs: make logs-mv"
    fi
  fi
fi
```
