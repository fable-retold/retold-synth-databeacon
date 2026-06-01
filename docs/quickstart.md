# Quick Start

Get Retold SynthDatabeacon running and serving synthetic records in a few minutes.

## Prerequisites

- **Node.js 20+** -- the Docker image is built on `node:20-slim`; Node 18 or later is expected to work but 20 is the tested baseline.
- **npm** -- included with Node.js.

## Installation

Install as a local dependency:

```sh
npm install retold-synth-databeacon
```

Or install globally so the `retold-synth-databeacon` command is available everywhere:

```sh
npm install -g retold-synth-databeacon
```

## First Run

Start the server with default settings:

```sh
retold-synth-databeacon
```

The beacon starts on port **8390** (chosen to avoid colliding with retold-databeacon's 8389 in shared compose stacks) and loads the two bundled specs from `source/specs/`. You will see output similar to:

```
Retold SynthDatabeacon initializing...
SynthDatabeacon: loaded 2 spec(s) from <bundled source/specs>: [customers-demo-v1, industrial-supply-v1]
Retold SynthDatabeacon running on port 8390
Discovery: http://localhost:8390/synth/specs
Health:    http://localhost:8390/synth/health
```

Override the port and the spec directory with flags:

```sh
retold-synth-databeacon --port 9000 --spec-dir /path/to/my/specs
```

## Verifying the Beacon

Check liveness:

```sh
curl http://localhost:8390/synth/health
```

```json
{ "Status": "OK", "Service": "retold-synth-databeacon" }
```

List the registered specs and their entities:

```sh
curl http://localhost:8390/synth/specs
```

```json
[
	{
		"Name": "customers-demo-v1",
		"Description": "Tiny two-entity demo spec used for smoke-testing the synth beacon's meadow-shape REST surface.",
		"EntityCount": 2,
		"Entities":
		[
			{ "Entity": "Company",  "Count": 25,  "FieldCount": 9 },
			{ "Entity": "Customer", "Count": 100, "FieldCount": 12 }
		]
	}
]
```

Inspect one entity's full definition:

```sh
curl http://localhost:8390/synth/specs/customers-demo-v1/entity/Customer
```

## Pulling Records

The data routes mirror meadow's standard endpoint shape. The entity segment is plural (a trailing `s` is stripped on lookup), matching the meadow convention.

### Paginated Slice

The pattern is `/1.0/:specName/:entityPlural/:offset/:count`:

```sh
# First 100 Customer rows from the demo spec
curl http://localhost:8390/1.0/customers-demo-v1/Customers/0/100
```

The response is a JSON array of records:

```json
[
	{
		"IDCustomer": 1,
		"GUIDCustomer": "...",
		"FirstName": "...",
		"LastName": "...",
		"FullName": "...",
		"Email": "...",
		"Phone": "...",
		"City": "...",
		"StateCode": "...",
		"IDCompany": 7,
		"TenureYears": 12,
		"CreatedDate": "2023-... T...Z"
	}
]
```

Offset and count are clamped against the entity's `Count`, so asking for `(950, 100)` on a 1000-row entity returns the last 50 rows rather than an error.

### Row Count

`/Count` returns a bare integer, matching meadow:

```sh
curl http://localhost:8390/1.0/customers-demo-v1/Customers/Count
```

```
100
```

### Default Page

Hitting the entity without offset/count returns the first page (up to 100 rows):

```sh
curl http://localhost:8390/1.0/customers-demo-v1/Customers
```

### Determinism in Practice

Fetch the same slice twice and the records are identical. Fetch overlapping pages and the shared rows match:

```sh
curl http://localhost:8390/1.0/customers-demo-v1/Customers/0/100   # rows 0-99
curl http://localhost:8390/1.0/customers-demo-v1/Customers/50/50   # rows 50-99, identical to the tail above
```

## Configuration

Configuration is resolved with this precedence (highest first): CLI flags, then `SYNTHBEACON_*` environment variables, then built-in defaults.

### CLI Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--port`, `-p <port>` | `8390` | API server port |
| `--spec-dir`, `-s <path>` | bundled `source/specs` | Directory of `*.json` spec files |
| `--log`, `-l [path]` | -- | Write log output to a file (a timestamped path is generated if none is given) |
| `--help`, `-h` | -- | Show usage |

### Environment Variables

| Variable | Equivalent | Description |
|----------|-----------|-------------|
| `SYNTHBEACON_PORT` | `--port` | API server port |
| `SYNTHBEACON_SPEC_DIR` | `--spec-dir` | Spec directory |
| `SYNTHBEACON_LOG_PATH` | `--log` | Log file path |
| `SYNTHBEACON_ULTRAVISOR_URL` | -- | If set, auto-connect to this Ultravisor coordinator on startup |
| `SYNTHBEACON_BEACON_NAME` | -- | Beacon name to register with (default `retold-synth-databeacon`) |
| `SYNTHBEACON_BEACON_PASSWORD` | -- | Auth password for the beacon connection |
| `SYNTHBEACON_MAX_CONCURRENT` | -- | Max concurrent work items (default `3`) |

Any secret-bearing variable also accepts a `_FILE` suffix (for example `SYNTHBEACON_BEACON_PASSWORD_FILE`), which sources the value from a file path. This follows the mysql/postgres image convention and works with Docker secrets and Kubernetes Secret mounts.

> The standalone CLI also honors the generic `PORT` environment variable as a fallback when `--port` and `SYNTHBEACON_PORT` are unset.

## Running with Docker

Build the image:

```sh
docker build -t retold-synth-databeacon .
```

Or use the packaged npm script:

```sh
npm run docker-build
```

Run it, mapping the default port:

```sh
docker run --rm -p 8390:8390 retold-synth-databeacon
```

The image is a multi-stage build on `node:20-slim` with no web UI bundle (SynthDatabeacon has no browser interface). It exposes port 8390 and ships a `HEALTHCHECK` that polls `/synth/health`.

To serve your own specs, mount a directory and point the beacon at it:

```sh
docker run --rm -p 8390:8390 \
	-v "$PWD/my-specs:/specs" \
	-e SYNTHBEACON_SPEC_DIR=/specs \
	retold-synth-databeacon
```

## Connecting to Ultravisor

SynthDatabeacon registers a `MeadowProxy` capability so consumers can read its records over the Ultravisor mesh. Set the coordinator URL and the beacon auto-connects on startup:

```sh
SYNTHBEACON_ULTRAVISOR_URL=ws://ultravisor:8080 \
SYNTHBEACON_BEACON_NAME=synth-demo \
SYNTHBEACON_BEACON_PASSWORD_FILE=/run/secrets/beacon_pw \
retold-synth-databeacon
```

On success you will see:

```
Auto-connecting to Ultravisor at ws://ultravisor:8080 as "synth-demo"...
Ultravisor auto-connect succeeded - registered as "synth-demo".
```

The beacon connects read-only (`AllowWrites: false`). Because the capability name matches retold-databeacon's, any mesh consumer that targets `MeadowProxy` -- such as retold-data-mapper's PullRecords -- works against this beacon with no changes. See [Architecture](architecture.md) for the details of the proxy path.

## Next Steps

- [Architecture](architecture.md) -- the services, the request flow, and the per-cell seeding model.
- [Specs Guide](specs.md) -- the bundled specs, the full `synth_*` function reference, and how to author your own.
