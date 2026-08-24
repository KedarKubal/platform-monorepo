# Platform Monorepo

Four independently-buildable projects — a Node feature-flag service, a Python
ETL pipeline, a Flutter design system, and a static analytics demo site —
wired together through one real integration point: **the toggle service acts
as a control plane, and its audit trail becomes a first-class data source.**

This isn't four repos sharing a `.git` folder. It's a demonstration of service
design, event-driven ETL extensibility, and cross-language client consumption,
all hanging off the same source of truth.

## Why this repo exists

Most portfolio monorepos prove you can nest directories. This one tries to
prove something narrower and more useful: that a small internal service
(feature flags) can be the seam a whole platform grows around — read by a
Python ETL pipeline, read by a static JS site, read by a Flutter component
catalog, and written to by three of those clients — without any of the four
projects knowing much about each other beyond one HTTP contract.

## Architecture

```
                        ┌─────────────────────────────┐
                        │   config-toggle-service      │
                        │   (Node/Express, control      │
                        │    plane for feature flags)  │
                        │                               │
                        │   flags.json  ── writes ──►   │
                        │   flag_audit.json ── writes ► │
                        └───────────┬───────────────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              │                     │                     │
       GET /api/flags/       GET /api/flags/audit    GET /api/flags/:key
       :key (poll)           ?since=<ts>              (fail-open read)
              │                     │                     │
              ▼                     ▼                     ▼
     ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
     │  flutter-dsm     │   │ migration-       │   │ migration-      │
     │  (Widgetbook)    │   │ platform (ETL)   │   │ platform (CLI)  │
     │                  │   │                  │   │                 │
     │ FlagService      │   │ FlagAuditExtractor│  │ FlagsClient     │
     │ polls a variant  │   │ ingests every    │   │ gates strict-   │
     │ flag on a timer  │   │ create/update/   │   │ email-validation│
     │ to live-switch a │   │ toggle/delete    │   │ at runtime,     │
     │ DsmButton preview│   │ into a queryable │   │ fails open if   │
     │                  │   │ FlagChangeAudit  │   │ toggle-service  │
     │                  │   │ table            │   │ is unreachable  │
     └──────────────────┘   └──────────────────┘   └─────────────────┘

     ┌────────────────────────────────────────────┐
     │  adobe-analytics-demo (static site)          │
     │  fetchFlag('cart-tracking-enabled') gates    │
     │  _satellite.track() calls in datalayer.js    │
     └───────────────────────────────────────────────┘
```

Four consumers, one control plane, two different failure philosophies by
design: the **runtime read path** (`FlagsClient`, the Adobe demo site, the
Widgetbook poller) fails open — a flag service outage should never take down
a migration run, a demo site, or a component catalog. The **audit ETL path**
(`FlagAuditExtractor`) fails loud on purpose — silently dropping an audit
batch would mean the pipeline reports success while quietly losing data,
which contradicts the "quarantine, don't crash" philosophy the ETL pipeline
is built around everywhere else.

## Repo layout

```
platform-monorepo/
├── services/
│   └── config-toggle-service/    Node/Express feature-flag control plane
├── apps/
│   ├── migration-platform/       Python ETL: legacy data + flag audit trail → normalized schema
│   └── adobe-analytics-demo/     Static cart-tracking demo, flag-gated
├── packages/
│   └── flutter-dsm/              Melos workspace: design tokens, components, Widgetbook catalog
├── docker-compose.yml            Root — wires postgres + toggle-service + demo site together
├── Taskfile.yml                  Root — cross-project test/build commands
└── README.md                     This file
```

Each project keeps its own dependency manifest (`package.json`,
`requirements.txt`, `pubspec.yaml`, `melos.yaml`) exactly where it always
lived — these are relocated directories with preserved git history, not a
merged build system.

## The four coupling points

1. **Audit endpoint** (`services/config-toggle-service`) — every flag
   create/update/toggle/delete appends an entry to `flag_audit.json` via the
   same write-queue + atomic-rename pattern the flag store itself uses.
   Exposed at `GET /api/flags/audit?since=<ISO timestamp>` and
   `GET /api/flags/:key/history`.
2. **Flag-audit ETL source** (`apps/migration-platform`) —
   `FlagAuditExtractor` polls that endpoint, transforms and validates the
   entries, and upserts them into a `FlagChangeAudit` table alongside the
   pipeline's other sources — proving the "extend by adding an `Extractor`
   subclass" seam the ETL's own README documents actually holds up for a
   real external source, not just CSVs.
3. **Runtime flag read** (`apps/migration-platform`) — a thin `FlagsClient`
   gates a stricter email-validation rule behind a live flag, caching for the
   run and failing open if the service is unreachable.
4. **Live variant polling** (`packages/flutter-dsm/packages/dsm_widgetbook`)
   — a Widgetbook use case polls a `component-preview-variant` flag on a
   timer to live-switch which `DsmButton` variant is shown, without a
   rebuild. The `http` dependency is scoped to `dsm_widgetbook` only —
   `dsm_components` stays free of anything network-related.

(The Adobe demo site's flag-gated cart tracking predates this list slightly
in the commit history but follows the same pattern — see its own README.)

## Getting started

```bash
# Install Melos once, for the flutter-dsm workspace
dart pub global activate melos
cd packages/flutter-dsm && melos bootstrap && cd ../..

# Bring up postgres + toggle-service + the demo site
task up

# Run every project's test suite
task test:all
```

`migration-platform` isn't a long-running compose service — it's a CLI
invoked via `task test:migration-platform` or directly with
`python -m src.orchestration.cli run`, pointed at `TOGGLE_SERVICE_URL` for
the flag-gated paths.

See each project's own README for full setup, environment variables, and
design rationale:

- [`services/config-toggle-service/README.md`](services/config-toggle-service/README.md)
- [`apps/migration-platform/README.md`](apps/migration-platform/README.md)
- [`packages/flutter-dsm/README.md`](packages/flutter-dsm/README.md)
- [`apps/adobe-analytics-demo/README.md`](apps/adobe-analytics-demo/README.md)

## Known gaps (honest, not hidden)

- **No CI workflow yet.** The original plan called for a path-filtered
  GitHub Actions setup (per-project jobs, plus an integration job gated on
  changes to any of the four coupling points) — not built yet. `task
  test:all` is the current substitute.
- **`task test:migration-platform` assumes a `.venv`.** It runs `source
  .venv/bin/activate && pytest -v`; adjust to your own environment manager
  (conda, etc.) if you don't use a venv at that path.
- **`getFlagHistory` returns an empty history for pre-seeded flags.** This is
  expected, documented behavior — pre-seeded flags were never written
  through the audited create/update/toggle path, so there's nothing in
  `flag_audit.json` for them yet — not a bug.
- **No `lint:migration-platform` task yet**, despite `ruff` being a
  dependency — added for local use, not yet wired into the Taskfile.
