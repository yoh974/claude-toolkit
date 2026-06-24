---
name: add-rattachement-entity
description: "Generates all rattachement artifacts for a new entity: value object, label utility, Added/Edited projection handlers, and updates the domain index.ts"
argument-hint: "<entity-kebab> <domain-module> <ITEM_TYPE> \"<French label prefix>\" — e.g.: economic-situation social SOCIAL \"Situation économique\""
allowed-tools: ["Bash", "Glob", "Grep", "Read", "Write", "Edit", "Agent"]
---

# Add Rattachement Entity

Generate all rattachement artifacts for a new entity following the established pattern from `economic-situation`.

**Arguments:** "$ARGUMENTS"

Parse arguments:
- `ENTITY_KEBAB` — entity name in kebab-case (e.g. `economic-situation`)
- `DOMAIN_MODULE` — source domain module alias, lowercase (e.g. `social`, `health`, `work`)
- `ITEM_TYPE` — rattachement itemType constant in SCREAMING_SNAKE_CASE (e.g. `SOCIAL`, `CLINICAL_HEALTH`, `WORK`)
- `FRENCH_LABEL_PREFIX` — French display name for the label value object (e.g. `Situation économique`)

Derive from these:
- `EntityPascal` = PascalCase of ENTITY_KEBAB (e.g. `EconomicSituation`)
- `ENTITY_SNAKE` = SCREAMING_SNAKE_CASE of ENTITY_KEBAB (e.g. `ECONOMIC_SITUATION`)
- `entityCamel` = camelCase of ENTITY_KEBAB (e.g. `economicSituation`)

---

## Step 1 — Discover domain events

Find the domain events for this entity:

```bash
find src/$DOMAIN_MODULE/domain/events/$ENTITY_KEBAB -type f -name "*.event.ts" 2>/dev/null || \
find src/$DOMAIN_MODULE/domain/events -type f -name "$ENTITY_KEBAB-*.event.ts" 2>/dev/null
```

Read each event file to understand:
- The event class names (e.g. `EconomicSituationAdded`, `EconomicSituationEdited`)
- The payload type names and fields (especially: the entity ID field name, `individualRecordId`, date fields, label fields)
- The import paths

Look specifically for:
- An **Added** event (the primary creation event)
- An **Edited** event (the update event)
- Any other events (Imported, Deleted, etc.) — note them but only generate handlers for Added and Edited by default

Also check the Prisma schema to know the exact model name for this entity:
```bash
grep -r "model.*$EntityPascal\b\|model.*$entityCamel\b" prisma/schema/ --include="*.prisma" -l
```
Read the relevant schema file to find: model name, available fields (especially date fields, label fields, version field).

---

## Step 2 — Determine label strategy

Based on the event payload fields and Prisma model fields, determine which label strategy applies:

**Strategy A — FuzzyDate fields** (like `economic-situation`):
- Entity has `startDate` and/or `endDate` as `FuzzyDate` fields in payload and DB
- Label built from formatted date parts
- Uses `FuzzyDate.fromPlainObject().toFrenchFormat()` and `FuzzyDate.fromDb().toFrenchFormat()`

**Strategy B — Simple text/referentiel label** (like `pathology`):
- Entity has a label string field in the event payload
- Label built directly from that field

**Strategy C — No extra data** (entity type is self-descriptive):
- Label is just the PREFIX

Choose the strategy that matches the available fields. When in doubt, prefer Strategy A if there are date fields.

---

## Step 3 — Generate the Value Object

Create file:
`src/rattachement/domain/value-objects/$DOMAIN_MODULE/$ENTITY_KEBAB-rattachement-label.vo.ts`

**Template (adapt `fromParts` signature to match label strategy):**

```typescript
export class {EntityPascal}RattachementLabel {
  private static readonly PREFIX = '{FRENCH_LABEL_PREFIX}';

  private constructor(private readonly value: string) {}

  // Adapt parameters based on strategy:
  // Strategy A: fromParts(startDate?: string, endDate?: string)
  // Strategy B: fromParts(labelText?: string, startDate?: string, endDate?: string)
  // Strategy C: fromParts()
  static fromParts(/* parameters */): {EntityPascal}RattachementLabel {
    const parts = [
      {EntityPascal}RattachementLabel.PREFIX,
      // ... add param?.trim() for each non-empty parameter
    ].filter((part): part is string => Boolean(part));

    return new {EntityPascal}RattachementLabel(parts.join(' - '));
  }

  getValue(): string {
    return this.value;
  }
}
```

---

## Step 4 — Generate the Label Utility

Create file:
`src/rattachement/infrastructure/projections/$DOMAIN_MODULE/$ENTITY_KEBAB/$ENTITY_KEBAB-rattachement-label.utils.ts`

**Template for Strategy A (FuzzyDate):**

```typescript
import { Prisma } from '@prisma/client';
import { {EntityPascal}RattachementLabel } from '@rattachement/domain/value-objects/$DOMAIN_MODULE/$ENTITY_KEBAB-rattachement-label.vo';
import { FuzzyDate, FuzzyDatePlainObject } from '@shared/domain/value-object/fuzzy-date.vo';

export const {ENTITY_SNAKE}_TYPE = '{ENTITY_SNAKE}';

export async function build{EntityPascal}RattachementLabel(
  payload: {
    {entityCamel}Id: string;
    startDate?: FuzzyDatePlainObject;
    endDate?: FuzzyDatePlainObject;
  },
  transaction: Prisma.TransactionClient,
): Promise<string> {
  const {entityCamel} = await transaction.{entityCamel}.findUnique({
    where: { id: payload.{entityCamel}Id },
    select: { startDate: true, endDate: true },
  });

  const startDate = payload.startDate
    ? FuzzyDate.fromPlainObject(payload.startDate).toFrenchFormat()
    : {entityCamel}?.startDate
      ? FuzzyDate.fromDb({entityCamel}.startDate).toFrenchFormat()
      : '';

  const endDate = payload.endDate
    ? FuzzyDate.fromPlainObject(payload.endDate).toFrenchFormat()
    : {entityCamel}?.endDate
      ? FuzzyDate.fromDb({entityCamel}.endDate).toFrenchFormat()
      : '';

  return {EntityPascal}RattachementLabel.fromParts(startDate, endDate).getValue();
}
```

**Template for Strategy B (text label):**

```typescript
import { Prisma } from '@prisma/client';
import { {EntityPascal}RattachementLabel } from '@rattachement/domain/value-objects/$DOMAIN_MODULE/$ENTITY_KEBAB-rattachement-label.vo';

export const {ENTITY_SNAKE}_TYPE = '{ENTITY_SNAKE}';

export async function build{EntityPascal}RattachementLabel(
  payload: {
    {entityCamel}Id: string;
    labelField?: string; // adapt field name
  },
  transaction: Prisma.TransactionClient,
): Promise<string> {
  // If label not in payload, fetch from DB
  const labelText = payload.labelField ?? (
    await transaction.{entityCamel}.findUnique({
      where: { id: payload.{entityCamel}Id },
      select: { labelField: true },
    })
  )?.labelField ?? undefined;

  return {EntityPascal}RattachementLabel.fromParts(labelText).getValue();
}
```

Adapt the template to the actual payload and DB field names discovered in Step 1 and Step 2.

---

## Step 5 — Generate the Added Projection Handler

Create file:
`src/rattachement/infrastructure/projections/$DOMAIN_MODULE/$ENTITY_KEBAB/$ENTITY_KEBAB-added-rattachement-item.projection.handler.ts`

Use this exact structure (adapt event imports and payload field names):

```typescript
import { EventStoreService } from '@core/event-store/event-store.service';
import { DomainEvent } from '@core/events/domain-event';
import { MongoBulkWriterService } from '@core/mongo/mongo-bulk-writer.service';
import { PrismaService } from '@core/prisma/prisma.service';
import { BaseProjectionHandler } from '@core/projections/projection-handler.base';
import { Projection, ProjectionOptions } from '@core/projections/projection.decorator';
import { Logger } from '@nestjs/common';
import { Prisma } from '@prisma/client';
import {
  {EntityPascal}Added,
  {EntityPascal}AddedPayload,
} from '@$DOMAIN_MODULE/domain/events/$ENTITY_KEBAB/$ENTITY_KEBAB-added.event';
import { replaceByIndividualRecordIds } from '../../shared/rattachement-batch-replace.utils';
import {
  build{EntityPascal}RattachementLabel,
  {ENTITY_SNAKE}_TYPE,
} from './$ENTITY_KEBAB-rattachement-label.utils';

@Projection({EntityPascal}Added)
export class {EntityPascal}AddedRattachementItemProjectionHandler extends BaseProjectionHandler {
  protected logger = new Logger({EntityPascal}AddedRattachementItemProjectionHandler.name);

  constructor(
    protected readonly prisma: PrismaService,
    protected readonly eventStore: EventStoreService,
    private readonly mongoBulkWriter: MongoBulkWriterService,
  ) {
    super(prisma, eventStore);
  }

  protected async isEventAlreadyProjected(
    event: DomainEvent<{EntityPascal}AddedPayload>,
    transaction: Prisma.TransactionClient,
  ): Promise<boolean> {
    const rattachementItem = await transaction.{entityCamel}.findUnique({
      where: { id: event.aggregate_id },
      select: { version: true },
    });

    if (!rattachementItem) {
      return false;
    }

    return event.version <= rattachementItem.version;
  }

  protected async project(
    event: DomainEvent<{EntityPascal}AddedPayload>,
    transaction: Prisma.TransactionClient,
  ): Promise<void> {
    const payload = event.payload;

    this.logger.verbose(
      `Projecting {EntityPascal}Added into rattachement_items ({entityCamel}Id="${payload.{entityCamel}Id}")`,
    );

    const label = await build{EntityPascal}RattachementLabel(
      { {entityCamel}Id: payload.{entityCamel}Id, /* add date/label fields from payload */ },
      transaction,
    );

    const data = {
      type: {ENTITY_SNAKE}_TYPE,
      itemType: '$ITEM_TYPE',
      label,
      individualRecordId: payload.individualRecordId,
      status: 'ACTIVE',
      lastEventVersion: event.version,
      createdAt: new Date(event.created_at),
      updatedAt: new Date(event.created_at),
    };

    await transaction.rattachementItem.upsert({
      where: { id: payload.{entityCamel}Id },
      create: { id: payload.{entityCamel}Id, ...data },
      update: data,
    });
  }

  protected async projectBatch(
    events: DomainEvent<{EntityPascal}AddedPayload>[],
    _transaction: Prisma.TransactionClient,
    _options?: ProjectionOptions,
  ): Promise<void> {
    const items: Record<string, unknown>[] = [];
    const individualRecordIds = new Set<string>();

    for (const event of events) {
      const payload = event.payload;
      if (!payload.individualRecordId) continue;
      individualRecordIds.add(payload.individualRecordId);

      const label = await build{EntityPascal}RattachementLabel(
        { {entityCamel}Id: payload.{entityCamel}Id, /* add date/label fields from payload */ },
        this.prisma,
      );

      items.push({
        id: payload.{entityCamel}Id,
        type: {ENTITY_SNAKE}_TYPE,
        itemType: '$ITEM_TYPE',
        label,
        individualRecordId: payload.individualRecordId,
        status: 'ACTIVE',
        lastEventVersion: event.version,
        createdAt: new Date(event.created_at),
        updatedAt: new Date(event.created_at),
      });
    }

    await replaceByIndividualRecordIds(
      this.mongoBulkWriter,
      'rattachement_items',
      items,
      individualRecordIds,
      { type: {ENTITY_SNAKE}_TYPE },
    );
  }
}
```

**Important:** Replace `{entityCamel}Id` with the actual ID field name from the event payload (may differ — check the payload type from Step 1).

---

## Step 6 — Generate the Edited Projection Handler

Create file:
`src/rattachement/infrastructure/projections/$DOMAIN_MODULE/$ENTITY_KEBAB/$ENTITY_KEBAB-edited-rattachement-item.projection.handler.ts`

```typescript
import { EventStoreService } from '@core/event-store/event-store.service';
import { DomainEvent } from '@core/events/domain-event';
import { PrismaService } from '@core/prisma/prisma.service';
import { BaseProjectionHandler } from '@core/projections/projection-handler.base';
import { Projection, ProjectionOptions } from '@core/projections/projection.decorator';
import { Logger } from '@nestjs/common';
import { Prisma } from '@prisma/client';
import {
  {EntityPascal}Edited,
  {EntityPascal}EditedPayload,
} from '@$DOMAIN_MODULE/domain/events/$ENTITY_KEBAB/$ENTITY_KEBAB-edited.event';
import {
  build{EntityPascal}RattachementLabel,
  {ENTITY_SNAKE}_TYPE,
} from './$ENTITY_KEBAB-rattachement-label.utils';

@Projection({EntityPascal}Edited)
export class {EntityPascal}EditedRattachementItemProjectionHandler extends BaseProjectionHandler {
  protected logger = new Logger({EntityPascal}EditedRattachementItemProjectionHandler.name);

  constructor(
    protected readonly prisma: PrismaService,
    protected readonly eventStore: EventStoreService,
  ) {
    super(prisma, eventStore);
  }

  protected async isEventAlreadyProjected(
    event: DomainEvent<{EntityPascal}EditedPayload>,
    transaction: Prisma.TransactionClient,
  ): Promise<boolean> {
    const rattachementItem = await transaction.{entityCamel}.findUnique({
      where: { id: event.aggregate_id },
      select: { version: true },
    });

    if (!rattachementItem) {
      return false;
    }

    return event.version <= rattachementItem.version;
  }

  protected async project(
    event: DomainEvent<{EntityPascal}EditedPayload>,
    transaction: Prisma.TransactionClient,
  ): Promise<void> {
    const payload = event.payload;

    this.logger.verbose(
      `Projecting {EntityPascal}Edited into rattachement_items ({entityCamel}Id="${payload.{entityCamel}Id}")`,
    );

    const label = await build{EntityPascal}RattachementLabel(
      { {entityCamel}Id: payload.{entityCamel}Id, /* add date/label fields from payload */ },
      transaction,
    );

    const data = {
      type: {ENTITY_SNAKE}_TYPE,
      itemType: '$ITEM_TYPE',
      label,
      individualRecordId: payload.individualRecordId,
      status: 'ACTIVE',
      lastEventVersion: event.version,
      createdAt: new Date(event.created_at),
      updatedAt: new Date(event.created_at),
    };

    await transaction.rattachementItem.upsert({
      where: { id: payload.{entityCamel}Id },
      create: { id: payload.{entityCamel}Id, ...data },
      update: data,
    });
  }

  protected async projectBatch(
    events: DomainEvent<{EntityPascal}EditedPayload>[],
    transaction: Prisma.TransactionClient,
    options?: ProjectionOptions,
  ): Promise<void> {
    for (const event of events) {
      if (!options?.skipIdempotencyCheck) {
        const alreadyProjected = await this.isEventAlreadyProjected(event, transaction);
        if (alreadyProjected) continue;
      }
      await this.project(event, transaction);
    }
  }
}
```

---

## Step 7 — Update the domain index.ts

Find the index file for this domain module's projections:
`src/rattachement/infrastructure/projections/$DOMAIN_MODULE/index.ts`

Add the two new handlers:
1. Import `{EntityPascal}AddedRattachementItemProjectionHandler` from `./$ENTITY_KEBAB/$ENTITY_KEBAB-added-rattachement-item.projection.handler`
2. Import `{EntityPascal}EditedRattachementItemProjectionHandler` from `./$ENTITY_KEBAB/$ENTITY_KEBAB-edited-rattachement-item.projection.handler`
3. Add both to the exported array

---

## Step 8 — Verify fuzzy-date.vo.ts

Check `src/shared/domain/value-object/fuzzy-date.vo.ts` has these methods (required by the label utilities):
- `static fromDb(obj: { day: string | null; month: string | null; year: string | null }): FuzzyDate`
- `toFrenchFormat(): string` — returns `dd/mm/yyyy` format

If either method is missing, add it following the existing pattern in the file.

--- 

## Step 9 — Type-check

Run a type check to validate the generated files:

```bash
npx tsc --noEmit 2>&1 | head -50
```

Fix any type errors before finishing.

---

## Output

When done, report:
- Files created (with paths)
- Files modified (with what changed)
- Any type errors found and fixed
- Any assumptions made about payload field names (so the user can verify)
