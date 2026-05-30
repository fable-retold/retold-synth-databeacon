# Specs Guide

A **spec** is the declarative input that defines what SynthDatabeacon generates. This guide covers the spec file format, the two bundled specs, the full `synth_*` expression function reference, and how to author your own.

## Spec File Format

A spec is a single JSON file describing one or more entities. Spec files live in the spec directory (the bundled `source/specs/`, or whatever `--spec-dir` / `SYNTHBEACON_SPEC_DIR` points at) and are loaded at startup.

```json
{
	"Name": "industrial-supply-v1",
	"GlobalSeed": "industrial-supply-v1",
	"Description": "A 14-table industrial-supply demo dataset",
	"Entities":
	[
		{
			"Entity": "Company",
			"Count": 50,
			"Fields":
			[
				{ "Column": "IDCompany", "Expression": "RecordIndex + 1" },
				{ "Column": "Name",      "Expression": "synth_company()" }
			]
		}
	]
}
```

### Spec fields

| Field | Required | Purpose |
|-------|----------|---------|
| `Name` | yes | Unique spec name; the first segment of the REST path (`/1.0/:specName/...`). |
| `Entities` | yes | Non-empty array of entity definitions. |
| `GlobalSeed` | no | Seed used for generation; defaults to `Name`. The bundled specs set it equal to `Name`. |
| `Description` | no | Free text shown by the discovery routes. |

### Entity fields

| Field | Required | Purpose |
|-------|----------|---------|
| `Entity` | yes | Entity name; the (singular) entity segment of the REST path. Must be unique within the spec. |
| `Count` | yes | Number of rows the entity can produce; also what `/Count` returns. Must be a non-negative integer. |
| `Fields` | yes | Non-empty array of `{ Column, Expression }` objects. |
| `GlobalSeed` | no | Per-entity seed override (rare; isolates one entity's stream for fixtures). |

### Field objects

Each field is a `Column` name and an `Expression`. The expression is evaluated by fable's ExpressionParser, extended with the `synth_*` functions. Three rules govern expressions:

1. **Double-quote every string literal.** Single quotes are tokenized as math and break (`"2020-01-01"`, not `'2020-01-01'`). See the note below.
2. **Use `CONCAT(...)` to join strings.** The `+` operator is strictly numeric.
3. **Reference earlier columns by name.** Fields resolve top-to-bottom; a later field can read any column declared above it, plus the row metadata symbols `RecordIndex`, `Entity`, and `GlobalSeed`.

An empty or whitespace-only expression yields an empty-string column. An expression that throws degrades to `null` (and logs a warning) rather than failing the whole request.

> **Why double quotes?** fable's ExpressionParser does not treat single quotes as string boundaries. `'2020-01-01'` is parsed as the arithmetic `2020 - 01 - 01`, which collapses to the number `-1`. Always use `"..."` for date ranges, pipe-delimited picker values, and cross-entity reference names.

## The `synth_*` Function Reference

Each `synth_*` function returns one deterministically-seeded value. Arguments are positional scalars; list-valued arguments (pickers) are passed as pipe-delimited strings because arrays do not survive the parser cleanly.

### Strings -- people, places, business

| Function | Returns |
|----------|---------|
| `synth_firstName()` | A first name. |
| `synth_lastName()` | A last name. |
| `synth_fullName()` | A full name. |
| `synth_email()` | An email address at `example.com`. |
| `synth_phone()` | A formatted US phone number. |
| `synth_address()` | A street address. |
| `synth_city()` | A city name. |
| `synth_state()` | A full state name. |
| `synth_stateCode()` | A 2-letter state code. |
| `synth_country()` | A full country name. |
| `synth_countryCode()` | An ISO country code. |
| `synth_postalCode()` | A US ZIP code. |
| `synth_company()` | A company name (a made-up capitalized word plus a suffix such as `Inc.`, `LLC`, `Industries`). |
| `synth_profession()` | A profession label. |
| `synth_department()` | A pick from a curated list of org-chart departments (Engineering, Sales, Finance, ...). |

### Identifiers

| Function | Returns |
|----------|---------|
| `synth_guid()` | A UUID. |

### Numerics

These are the deterministic, seeded alternatives to fable's built-in `RANDOMINTEGER` / `RANDOMFLOATBETWEEN` (which use `Math.random` and are *not* reproducible). Use the `synth_*` forms when you need stable output.

| Function | Returns |
|----------|---------|
| `synth_integer(min, max)` | A seeded integer in `[min..max]`. |
| `synth_floating(min, max)` | A seeded float in `[min..max]`, fixed to 4 decimal places. Wrap with `ROUND()` for currency precision. |

### Pickers

| Function | Returns |
|----------|---------|
| `synth_pickone("a\|b\|c")` | A uniform pick from the pipe-delimited values. |
| `synth_pickWeighted("a\|b\|c", "50\|30\|20")` | A weighted pick; weights are pipe-delimited and aligned to the values (short tails default to weight 1, extras are dropped). |

### Dates

Dates are returned as ISO 8601 strings, which is what meadow's DateTime columns expect on the wire.

| Function | Returns |
|----------|---------|
| `synth_dateBetween("YYYY-MM-DD", "YYYY-MM-DD")` | An ISO date within the inclusive range. |
| `synth_dateRecent(daysBack)` | An ISO date within the last N days (default 30). |

### Long text

| Function | Returns |
|----------|---------|
| `synth_sentence()` | A 5-12 word sentence. |
| `synth_paragraph(sentences)` | An N-sentence paragraph (default 3). |

### Relational

| Function | Returns |
|----------|---------|
| `synth_referenceTo("TargetEntity", targetCount)` | A deterministic foreign key -- a seeded integer in `[1..targetCount]`. |

`synth_referenceTo` is the keystone primitive for relational data. The `targetCount` argument should match the target entity's `Count` so the generated key always lands on a valid parent row. The target-entity *name* is documentary -- the function does not look the entity up; it simply produces an integer in range. (Deliberately undersizing the count is how the bundled spec injects orphan foreign keys; see fault injection below.)

### Fault injection

| Function | Returns |
|----------|---------|
| `synth_coinFlip(probability)` | `1` with the given probability, else `0`. |

`synth_coinFlip` is built to pair with a ternary so a fraction of records gets a degraded value -- deterministically, so the *same* records degrade on every run:

```javascript
// 5% of rows get an empty Email
synth_coinFlip(0.05) ? "" :: synth_email()

// 30% of lots carry an ExpirationDate, the rest are blank
synth_coinFlip(0.30) ? synth_dateBetween("2025-11-01", "2027-12-31") :: ""

// 5% of inventory lots are zero-quantity
synth_coinFlip(0.05) ? 0 :: synth_integer(1, 2500)
```

> The ExpressionParser ternary uses `::` (not `:`) to separate the branches, as the examples show.

### Built-in expression helpers

These are provided by fable's ExpressionParser, not by SynthDatabeacon, but the bundled specs rely on them:

- `CONCAT(...)` -- variadic string concatenation (use this instead of `+` for strings).
- `RecordIndex` -- the 0-based row index, exposed as a bare symbol. The common ID idiom is `RecordIndex + 1`.

## Bundled Specs

Two specs ship in `source/specs/` and load automatically.

### customers-demo-v1

A tiny two-entity spec for smoke-testing the meadow-shape surface. `GlobalSeed` is `customers-demo-v1`.

| Entity | Count | Notable columns |
|--------|-------|-----------------|
| `Company` | 25 | `IDCompany`, `Name` (`synth_company`), `Industry` (`synth_pickWeighted`), `FoundedDate` |
| `Customer` | 100 | `IDCustomer`, `FullName` (`CONCAT` of first/last), `Email` (5% nulled via `synth_coinFlip`), `IDCompany` (`synth_referenceTo("Company", 25)`) |

### industrial-supply-v1

A 14-entity dataset of roughly 46,000 records spanning Identity, Catalog, Inventory, Sales, and Manufacturing domains. It is large enough for meaningful aggregations, histograms, and joins, but generates in seconds. `GlobalSeed` is `industrial-supply-v1`.

| Entity | Count | Fields |
|--------|-------|--------|
| `Company` | 50 | 14 |
| `Department` | 250 | 9 |
| `User` | 1000 | 18 |
| `ProductCategory` | 30 | 8 |
| `Product` | 1000 | 15 |
| `Material` | 500 | 13 |
| `Warehouse` | 25 | 14 |
| `InventoryLot` | 5000 | 13 |
| `MaterialLot` | 1500 | 11 |
| `Customer` | 1500 | 17 |
| `SalesOrder` | 5000 | 15 |
| `SalesOrderLine` | 25000 | 12 |
| `WorkOrder` | 1000 | 14 |
| `MaterialConsumption` | 5000 | 12 |

#### Relationships

The entities are wired together with `synth_referenceTo`, forming a connected graph across the five domains:

| Child column | References | Range |
|--------------|-----------|-------|
| `Department.IDCompany` | Company | 1..50 |
| `User.IDCompany` | Company | 1..50 |
| `User.IDDepartment` | Department | 1..250 |
| `Product.IDProductCategory` | ProductCategory | 1..30 |
| `Product.IDManufacturer` | Company | 1..50 |
| `Material.IDSupplier` | Company | 1..50 |
| `Warehouse.IDCompany` | Company | 1..50 |
| `InventoryLot.IDProduct` | Product | 1..1000 |
| `InventoryLot.IDWarehouse` | Warehouse | 1..25 |
| `MaterialLot.IDMaterial` | Material | 1..500 |
| `MaterialLot.IDWarehouse` | Warehouse | 1..25 |
| `Customer.*` | (see spec) | -- |
| `SalesOrder.IDCustomer` | Customer | 1..1500 |
| `SalesOrder.IDSalesRep` | User | 1..1000 |
| `SalesOrderLine.IDSalesOrder` | SalesOrder | 1..5000 |
| `SalesOrderLine.IDProduct` | Product | 1..1000 |
| `WorkOrder.IDProduct` | Product | 1..1000 |
| `WorkOrder.IDWarehouse` | Warehouse | 1..25 |
| `MaterialConsumption.IDWorkOrder` | WorkOrder | 1..1000 |
| `MaterialConsumption.IDMaterial` | Material | 1..500 |

#### Fault injection

The spec deliberately seeds dirty data so consumers can verify their pipelines handle it. The patterns are deterministic -- the same records carry the same defects on every run. Representative examples drawn from the spec:

- **Nullable contact fields** -- `User.Email` is empty ~5% of the time, `User.Phone` ~10%, via `synth_coinFlip` ternaries.
- **Zero-quantity inventory** -- `InventoryLot.QuantityOnHand` is `0` ~5% of the time.
- **Optional dates** -- `InventoryLot.ExpirationDate` is populated only ~30% of the time.

The spec's own description also notes ~1% orphan foreign keys (a reference whose target index can exceed the parent's real count). Inspect `source/specs/industrial-supply-v1.json` for the exact per-column probabilities.

## Authoring a Spec

1. **Create a JSON file** in a directory of your choice (or alongside the bundled specs).

	```json
	{
		"Name": "my-spec-v1",
		"GlobalSeed": "my-spec-v1",
		"Description": "My synthetic dataset",
		"Entities":
		[
			{
				"Entity": "Widget",
				"Count": 500,
				"Fields":
				[
					{ "Column": "IDWidget",  "Expression": "RecordIndex + 1" },
					{ "Column": "GUIDWidget","Expression": "synth_guid()" },
					{ "Column": "Name",      "Expression": "synth_company()" },
					{ "Column": "Price",     "Expression": "synth_floating(1, 999)" },
					{ "Column": "Category",  "Expression": "synth_pickone(\"A|B|C\")" },
					{ "Column": "IDOwner",   "Expression": "synth_referenceTo(\"User\", 100)" },
					{ "Column": "Created",   "Expression": "synth_dateBetween(\"2022-01-01\", \"2025-12-31\")" }
				]
			}
		]
	}
	```

2. **Point the beacon at it** -- either drop the file into `source/specs/`, or pass the directory:

	```sh
	retold-synth-databeacon --spec-dir /path/to/my/specs
	```

	With Docker, mount the directory and set `SYNTHBEACON_SPEC_DIR` (see the [Quick Start](quickstart.md)).

3. **Verify it loaded:**

	```sh
	curl http://localhost:8390/synth/specs
	curl http://localhost:8390/1.0/my-spec-v1/Widgets/0/50
	```

### Authoring checklist

- A unique `Name` and at least one entity with at least one field.
- A unique `Entity` name per entity, and a non-negative integer `Count`.
- Every string literal double-quoted; every string join via `CONCAT(...)`.
- IDs via `RecordIndex + 1`; foreign keys via `synth_referenceTo("Parent", parentCount)` with `parentCount` matching the parent's `Count`.
- Later fields may reference earlier columns by name.
- Set a `GlobalSeed` (commonly equal to `Name`) so output is reproducible across machines.

A spec that fails to parse or validate is logged and skipped at startup; the other specs still load. The validation rules are enforced by `SynthBeacon-SpecRegistry` (spec and entity shape) and `SynthBeacon-Evaluator` (field shape and `Count`).
