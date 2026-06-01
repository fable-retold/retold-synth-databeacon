Retold SynthDatabeacon
======================

An on-demand synthetic-record source for the Retold ecosystem. SynthDatabeacon serves a paginated REST surface that mirrors meadow's standard endpoint shape, but the records it returns are generated per request from declarative JSON specs rather than read from a real database.

Because it speaks the same meadow-shape `/1.0/:spec/:entity` surface and registers the same `MeadowProxy` beacon capability as [retold-databeacon](https://github.com/fable-retold/retold-databeacon), any consumer that already reads records over the Ultravisor mesh -- including retold-data-mapper's PullRecords -- can point at a synth spec as a drop-in data source with no code changes.

Generation is deterministic. Each cell is seeded from `globalSeed + entity + recordIndex + columnName`, so the same spec and seed always produce the same data, page after page, machine after machine. That makes SynthDatabeacon useful for demos, repeatable integration tests, and exercising downstream pipelines (aggregations, histograms, intersections) against substantial, dirty-by-design datasets without standing up a database.

## Features

- **Meadow-shape REST surface** -- paginated list, `/Count`, and default-page routes at `/1.0/:specName/:entityPlural`, the same shape meadow clients already speak.
- **Deterministic per-cell seeding** -- edit one column's expression and only that column shifts; every other cell stays bit-stable.
- **Declarative multi-entity specs** -- a single JSON file describes many related entities, their row counts, and per-column expressions.
- **`synth_*` expression functions** -- names, emails, addresses, dates, GUIDs, weighted pickers, and deterministic cross-entity foreign keys, all built on fable's ExpressionParser plus chance.js.
- **Fault injection** -- `synth_coinFlip(p)` paired with a ternary pins the same records to NULL / zero / orphan values on every run, so consumers can verify they handle dirty data.
- **MeadowProxy beacon capability** -- registers with an Ultravisor coordinator under the same capability name retold-databeacon ships, making it a drop-in source over the mesh.
- **Two bundled specs** -- `industrial-supply-v1` (14 entities, ~46K records) and `customers-demo-v1` (2 entities).
- **CLI, library, and Docker** -- run it as a global command, embed it as a `fable-serviceproviderbase` service, or deploy the multi-stage Docker image.

## Install

```sh
npm install retold-synth-databeacon
```

## Quick Start

```sh
# Start the beacon on the default port (8390) with the bundled specs
npx retold-synth-databeacon

# Discover the registered specs
curl http://localhost:8390/synth/specs

# Pull the first 100 Customer rows from the demo spec
curl http://localhost:8390/1.0/customers-demo-v1/Customers/0/100
```

See the [Quick Start guide](./docs/quickstart.md) for the full walkthrough.

## Documentation

Documentation lives in the [docs](./docs) folder:

- [Overview](./docs/README.md) -- what SynthDatabeacon is and how it fits the ecosystem
- [Quick Start](./docs/quickstart.md) -- get running in a few minutes
- [Architecture](./docs/architecture.md) -- services, request flow, and the determinism model
- [Specs Guide](./docs/specs.md) -- the bundled specs, the `synth_*` function reference, and how to author your own

## Related Modules

- [retold-databeacon](https://github.com/fable-retold/retold-databeacon) -- the real-database beacon SynthDatabeacon is a drop-in source for
- [meadow](https://github.com/fable-retold/meadow) -- the data access layer whose REST endpoint shape SynthDatabeacon mirrors
- [fable](https://github.com/fable-retold/fable) -- service container, configuration, and the ExpressionParser that evaluates specs
- [orator](https://github.com/fable-retold/orator) -- the HTTP server SynthDatabeacon serves through
- [ultravisor](https://github.com/stevenvelozo/ultravisor) -- the workflow / mesh coordinator a beacon registers with
- [ultravisor-beacon](https://github.com/stevenvelozo/ultravisor-beacon) -- the beacon-registration library that exposes the MeadowProxy capability

## License

MIT
