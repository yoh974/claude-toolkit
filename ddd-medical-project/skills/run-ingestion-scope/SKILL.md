---
name: run-ingestion-scope
description: "Lance un scope d'ingestion IR sur toutes les ingestions présentes en base. Par défaut: scope 'identity'."
argument-hint: "[scope?] — scope: identity|social|job-situation|affiliation|exposure-situation|clinical-health|lifestyle-and-addictions|all (défaut: identity)"
allowed-tools: ["Bash"]
---

# ig-scope-all — Lancer un scope IR pour toutes les ingestions

**Syntaxe :** `/ig-scope-all [scope?]`

Scopes disponibles : `identity`, `social`, `job-situation`, `affiliation`, `exposure-situation`, `clinical-health`, `lifestyle-and-addictions`, `all`

**Arguments :** "$ARGUMENTS"

## Étape 1 — Résoudre le scope

```bash
ARGS="$ARGUMENTS"
SCOPE="${ARGS:-identity}"
IR_API="http://ir.api.localhost/api/imports"
IR_DB="ir-module"

if [ "$SCOPE" = "all" ]; then
  SCOPES="identity social job-situation affiliation exposure-situation clinical-health lifestyle-and-addictions"
else
  SCOPES="$SCOPE"
fi

echo "Scope(s) à traiter : $SCOPES"
```

## Étape 2 — Récupérer toutes les ingestions depuis MongoDB

```bash
INGESTION_IDS=$(docker exec workspace-mongodb-1 mongosh --quiet --eval "
  db = db.getSiblingDB('$IR_DB');
  db['import.ingestions'].find({}, {_id:1}).toArray().forEach(d => print(d._id));
" 2>/dev/null | tr -d '\r')

if [ -z "$INGESTION_IDS" ]; then
  echo "ERREUR: aucune ingestion trouvée en base."
  exit 1
fi

COUNT=$(echo "$INGESTION_IDS" | wc -l)
echo "$COUNT ingestion(s) trouvée(s)"
echo ""
```

## Étape 3 — Lancer le scope pour chaque ingestion

```bash
for ID in $INGESTION_IDS; do
  for S in $SCOPES; do
    STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
      -X POST "$IR_API/$ID/process/$S" \
      -H "Content-Type: application/json" \
      -d '{}')
    echo "[$ID] $S -> $STATUS"
  done
done

echo ""
echo "=== Terminé ==="
```
