---
name: replay-reference-data-event
description: "Rejoue des événements ItemCreated du référentiel vers Kafka (consommés par questionnaire). Par item ID ou par recherche sur nom/code/catégorie."
argument-hint: "[--item-id <uuid>] [--search <keyword>] [--dry-run]"
allowed-tools: ["Bash"]
---

# replay-referentiel-item — Rejouer des ItemCreated vers Kafka

**Syntaxe :** `/replay-referentiel-item [--item-id <uuid>] [--search <keyword>] [--dry-run]`

- `--item-id`  : UUID d'un item précis à rejouer
- `--search`   : Mot-clé recherché dans le nom, le code ou les métadonnées de l'item (ex: `patholog`, `CIM`, `AMENAGEMENT`)
- `--dry-run`  : Affiche les items trouvés sans les rejouer

**Arguments :** "$ARGUMENTS"

## Étape 1 — Parser les arguments

```bash
ARGS="$ARGUMENTS"
ITEM_ID=""
SEARCH=""
DRY_RUN=""

while [ -n "$ARGS" ]; do
  TOKEN=$(echo "$ARGS" | awk '{print $1}')
  REST=$(echo "$ARGS" | cut -d' ' -f2-)
  [ "$REST" = "$TOKEN" ] && REST=""

  case "$TOKEN" in
    --dry-run)  DRY_RUN="1" ; ARGS="$REST" ;;
    --item-id)
      ITEM_ID=$(echo "$REST" | awk '{print $1}')
      ARGS=$(echo "$REST" | cut -d' ' -f2-)
      [ "$ARGS" = "$ITEM_ID" ] && ARGS=""
      ;;
    --search)
      SEARCH=$(echo "$REST" | awk '{print $1}')
      ARGS=$(echo "$REST" | cut -d' ' -f2-)
      [ "$ARGS" = "$SEARCH" ] && ARGS=""
      ;;
    *) ARGS="$REST" ;;
  esac
done

if [ -z "$ITEM_ID" ] && [ -z "$SEARCH" ]; then
  echo "❌ Argument requis : --item-id <uuid> ou --search <keyword>"
  echo "   Exemples :"
  echo "     /replay-referentiel-item --item-id 75e7a6bd-d0e3-409c-aee0-121f1896bf50"
  echo "     /replay-referentiel-item --search patholog"
  echo "     /replay-referentiel-item --search CIM --dry-run"
  exit 1
fi
```

## Étape 2 — Trouver et rejouer les items

```bash
COMPOSE_DIR="$WORKSPACE_DIR"

# Les valeurs sont injectées directement dans le script node (docker compose exec ne transmet pas les env vars)
docker compose -f "$COMPOSE_DIR/compose.yml" exec -T referentiel_service node -e "
const { PrismaClient } = require('@prisma/client');
const p = new PrismaClient();

const ITEM_ID = '$ITEM_ID';
const SEARCH  = '$SEARCH';
const DRY_RUN = $( [ -n '$DRY_RUN' ] && echo 'true' || echo 'false' );

async function main() {
  let where = { content_type: 'ItemCreated' };
  if (ITEM_ID) where.aggregate_id = ITEM_ID;

  const events = await p.outboxEvent.findMany({
    where,
    select: { aggregate_id: true, status: true, payload: true }
  });

  const filtered = SEARCH
    ? events.filter(e => {
        const p = e.payload;
        const haystack = ((p.name || '') + ' ' + (p.code || '') + ' ' + JSON.stringify(p.metadata || {})).toLowerCase();
        return haystack.includes(SEARCH.toLowerCase());
      })
    : events;

  if (filtered.length === 0) {
    console.log('⚠️  Aucun item trouvé' + (SEARCH ? ' pour : ' + SEARCH : ''));
    return;
  }

  console.log('📋 Items trouvés : ' + filtered.length);
  filtered.slice(0, 15).forEach(e =>
    console.log('   [' + e.status + '] ' + (e.payload.code || '?') + ' — ' + (e.payload.name || '?') + ' (' + e.aggregate_id + ')')
  );
  if (filtered.length > 15) console.log('   ... et ' + (filtered.length - 15) + ' autres');

  if (DRY_RUN) {
    console.log('');
    console.log('🔍 Dry-run — aucun event remis en PENDING.');
    return;
  }

  const ids = filtered.map(e => e.aggregate_id);
  const r = await p.outboxEvent.updateMany({
    where: { aggregate_id: { in: ids }, content_type: 'ItemCreated' },
    data: { status: 'PENDING', retry_count: 0, error_message: null, next_retry_at: null, processed_at: null }
  });

  console.log('');
  console.log('✅ ' + r.count + ' event(s) remis en PENDING — publication Kafka imminente.');
}

main().catch(e => { console.error('❌ ' + e.message); process.exit(1); }).finally(() => p.\$disconnect());
" 2>/dev/null
```

## Étape 3 — Vérifier la consommation côté questionnaire

```bash
if [ -z "$DRY_RUN" ]; then
  echo ""
  echo "⏳ Attente 4s puis vérification des logs questionnaire..."
  sleep 4

  COMPOSE_DIR="$WORKSPACE_DIR"
  LOGS=$(docker compose -f "$COMPOSE_DIR/compose.yml" logs questionnaire_service --since=10s 2>&1)

  COUNT=$(echo "$LOGS" | grep -c "ItemCreated" || true)
  if [ "$COUNT" -gt 0 ]; then
    echo "✅ $COUNT événement(s) ItemCreated détecté(s) dans les logs questionnaire."
    echo "$LOGS" | grep "ItemCreated" | tail -5
  else
    echo "⚠️  Aucun log ItemCreated détecté côté questionnaire dans les 10 dernières secondes."
    echo "   Vérifiez manuellement : make logs-questionnaire"
  fi
fi
```