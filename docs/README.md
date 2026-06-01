# Retold SynthDatabeacon

Retold SynthDatabeacon is an on-demand synthetic-record source. It serves a paginated REST surface shaped exactly like meadow's standard endpoints, but every record is generated per request from a declarative JSON spec instead of being read from a database. From the outside it looks like a real data source; behind it there is no database, just a spec and a seed.

SynthDatabeacon is built to be a drop-in source for [retold-databeacon](https://fable-retold.github.io/retold-databeacon/). It mirrors the meadow-shape `/1.0/:spec/:entity` surface and registers the same `MeadowProxy` Ultravisor capability, so any consumer that already pulls records over the mesh -- including retold-data-mapper's PullRecords -- can use a synth spec as a source unchanged.

## Why Synthetic Data

A real database is overkill when what you actually need is *plausible, repeatable* records:

- **Demos** -- show a pipeline or a UI populated with realistic-looking data without provisioning a database.
- **Repeatable tests** -- the same spec and seed produce byte-identical records every run, on every machine, so assertions stay stable.
- **Dirty-data exercises** -- bundled fault injection (nullable emails and phones, orphan foreign keys, zero-quantity inventory) lets consumers verify their pipelines survive imperfect input.
- **Scale without weight** -- the `industrial-supply-v1` spec produces roughly 46,000 records across 14 related tables, enough for meaningful aggregations and joins, while the process stays small and starts in seconds.

## How It Works

A spec is a JSON file describing one or more entities. Each entity has a row `Count` and a list of `Fields`, where every field is a `Column` name and an `Expression`. Expressions are evaluated by fable's ExpressionParser, extended with a family of `synth_*` functions that emit one realistic, deterministically-seeded value each.

When a request arrives for a slice of an entity, the Evaluator walks the requested rows one at a time. For each cell it derives a seed from `globalSeed + entity + recordIndex + columnName`, evaluates the expression, and assembles the row. The result is paginated JSON in the same shape a meadow endpoint would return.

```json
{
	"Entity": "Customer",
	"Count": 100,
	"Fields":
	[
		{ "Column": "IDCustomer",   "Expression": "RecordIndex + 1" },
		{ "Column": "GUIDCustomer", "Expression": "synth_guid()" },
		{ "Column": "FirstName",    "Expression": "synth_firstName()" },
		{ "Column": "LastName",     "Expression": "synth_lastName()" },
		{ "Column": "FullName",     "Expression": "CONCAT(FirstName, \" \", LastName)" },
		{ "Column": "Email",        "Expression": "synth_coinFlip(0.05) ? \"\" :: synth_email()" },
		{ "Column": "IDCompany",    "Expression": "synth_referenceTo(\"Company\", 25)" }
	]
}
```

## Determinism

Determinism is the property that makes synth data useful for testing. SynthDatabeacon seeds **per cell**, not per row, so:

1. Editing one column's expression only changes that column's output. Every other cell on every other row stays bit-stable.
2. Pagination is stable. Row 100 looks the same whether you fetch it in page `(0, 200)` or page `(100, 100)`.
3. Fault-injection ternaries pin the *same* records to NULL on every run -- reproducible pathological datasets, not random noise.

The seed precedence for a generation run is: an explicit seed argument, then the spec's `GlobalSeed`, then the entity name, then the literal `default`. The bundled specs set `GlobalSeed` equal to their `Name`.

## Endpoints

SynthDatabeacon exposes two route families. The meadow-shape routes are what consumers read records through; the `/synth/*` routes are for discovery and liveness.

| Method & Route | Purpose |
|----------------|---------|
| `GET /synth/health` | Liveness ping (`{ "Status": "OK" }`) |
| `GET /synth/specs` | List all registered specs with their entities |
| `GET /synth/specs/:specName` | One spec's full metadata |
| `GET /synth/specs/:specName/entity/:entityName` | One entity's definition |
| `GET /1.0/:specName/:entityPlural` | Meadow-shape default page (up to 100 rows) |
| `GET /1.0/:specName/:entityPlural/Count` | Bare integer row count |
| `GET /1.0/:specName/:entityPlural/:offset/:count` | Meadow-shape paginated slice |

The `:entityPlural` segment follows meadow's convention: the route handler first tries the literal segment, then strips one trailing `s` (so both `Customers` and an already-plural `Status` resolve).

## Bundled Specs

Two specs ship in `source/specs/` and load automatically at startup:

- **`industrial-supply-v1`** -- 14 entities (~46K records) across Identity, Catalog, Inventory, Sales, and Manufacturing domains, wired together with deterministic foreign keys and seeded fault injection.
- **`customers-demo-v1`** -- a tiny 2-entity spec (Company, Customer) for smoke-testing the meadow-shape surface.

See the [Specs Guide](specs.md) for the entity lists, the full `synth_*` function reference, and how to author your own spec.

## Documentation

- [Quick Start](quickstart.md) -- install, run, and pull records
- [Architecture](architecture.md) -- services, request flow, and the determinism plumbing
- [Specs Guide](specs.md) -- bundled specs, expression functions, and authoring

## Related Modules

- [retold-databeacon](https://fable-retold.github.io/retold-databeacon/) -- the real-database beacon SynthDatabeacon is a drop-in source for
- [meadow](https://fable-retold.github.io/meadow/) -- the data access layer whose REST endpoint shape SynthDatabeacon mirrors
- [fable](https://fable-retold.github.io/fable/) -- service container, configuration, and the ExpressionParser that evaluates specs
- [orator](https://fable-retold.github.io/orator/) -- the HTTP server SynthDatabeacon serves through
- [ultravisor](https://stevenvelozo.github.io/ultravisor/) -- the workflow / mesh coordinator a beacon registers with
- [ultravisor-beacon](https://stevenvelozo.github.io/ultravisor-beacon/) -- the beacon-registration library that exposes the MeadowProxy capability
