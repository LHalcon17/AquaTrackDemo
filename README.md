# AquaTrack ERP - Production Module

The Production & Manufacturing module of the AquaTrack ERP. It manages the full
production cycle from master data and planning through material requests,
issuance, shop-floor execution, finished goods, costing, and reporting.

The module deliberately owns **production only**. It consumes raw materials from
Inventory and posts costs to Finance; it does not implement inventory or
accounting itself. Those hand-offs happen through an event outbox.

**Live demo:** <https://aquatrack-production.onrender.com>

> The live site runs on a free Render instance. It sleeps after ~15 minutes of
> inactivity, so the first visit can take 30-60 seconds to wake up.

---

## What is in this repository

| File | Purpose |
|---|---|
| `server.js` | Zero-dependency HTTP server, REST API, business rules, storage layer |
| `seed.js` | Demo dataset used to populate an empty database |
| `config.json` | Shared Supabase URL + publishable anon key (committed on purpose) |
| `.env.example` | Template for a private local key (copy to `.env`, git-ignored) |
| `supabase/schema.sql` | The 13 `production_*` tables, indexes, and grants |
| `public/index.html` | UI shell: dashboard, 10 feature screens, all panels |
| `public/production-api.js` | Front-end logic: API calls, rendering, event wiring |
| `public/styles.css` | Styles, including the responsive/mobile rules |
| `backup.ps1` | Creates a timestamped zip snapshot in `backups/` |
| `package.json` | `npm start` and `npm run backup` scripts |
| `.gitignore` | Excludes `data.json`, `.env`, `node_modules/`, `backups/` |
| `.gitattributes` | LF line-ending normalization |
| `README.md` | This file |

There are no dependencies. `package.json` has no `dependencies` block, and the
runtime is Node's built-in `http` plus the global `fetch`.

Requirements: **Node.js 18 or newer**.

---

## Quick start

```bash
git clone https://github.com/LHalcon17/AquaTrackDemo.git
cd AquaTrackDemo
git checkout PRODUCTION
npm start
```

Then open <http://localhost:4000>.

`npm start` is just `node server.js`. The port comes from `PORT` and defaults
to `4000`.

If the database is empty, `seed.js` supplies the demo data on first boot: 5
products, 4 bills of materials, 3 production orders, material requests,
issuances, machines, and finished goods.

### Only one server at a time

Everyone shares the same Supabase database. Because each server keeps an
in-memory copy of the data and rewrites whole tables on every change, running
your local copy while the live Render site is also running will make the two
overwrite each other. Pick one:

- use the live link, or
- stop the Render service and run locally.

Never both during a demo.

---

## Configuration

Settings are resolved in this order, first match wins:

1. **Real environment variables**
2. **`.env`** - local only, git-ignored, may contain the private service key
3. **`config.json`** - committed, contains the shared URL + publishable anon key

```json
{
  "SUPABASE_URL": "https://<project-ref>.supabase.co",
  "SUPABASE_ANON_KEY": "sb_publishable_...",
  "PORT": 4000
}
```

| Variable | Where it belongs | Public? |
|---|---|---|
| `SUPABASE_URL` | `config.json` | Yes |
| `SUPABASE_ANON_KEY` | `config.json` | Yes - publishable key, protected by RLS |
| `SUPABASE_SERVICE_KEY` | `.env` only | **No** - bypasses all security |
| `PORT` | `config.json` / environment | Yes |

### Storage selection

- If `SUPABASE_URL` **and** a key are present, data lives in Supabase.
- Otherwise the server falls back to `data.json` and still runs offline.

The fallback exists so the project can start with no configuration at all. It
is a convenience for local experiments, not a second production database.

### The service key

The service role key bypasses Row Level Security. It must never appear in
`config.json`, `public/`, or any committed file. `.env` is git-ignored for this
reason. The committed anon key is safe to publish because it is only as
powerful as the RLS policies allow.

### Security posture (demo)

Row Level Security is **off** on the shared demo database, and the anon key is
committed, so anyone with the repo can read and write the demo data. That is an
accepted trade-off for a classroom demo: no installs, no setup, no secrets to
distribute.

**Never put real or personal data in this database.** If the project ever moves
past demo, the order is: enable RLS with policies first, then remove
`config.json` from the repository and distribute the config privately.

---

## Database schema

13 tables, all prefixed `production_` so they can share a Supabase project with
the other modules. Defined in `supabase/schema.sql`.

| Table | Holds |
|---|---|
| `production_products` | Product master: code, name, category, unit, price |
| `production_boms` | Bill of materials per product; items stored as JSON |
| `production_orders` | Production orders, stage list, output records |
| `production_plans` | Production schedule entries |
| `production_material_requests` | Material requests exploded from the BOM |
| `production_issuances` | Materials actually released to the floor |
| `production_machines` | Machine/line records and their status |
| `production_finished_goods` | Output that entered stock |
| `production_cost_rates` | Labor and machine cost per unit, per product |
| `production_cost_summaries` | Period cost summaries |
| `production_cost_postings` | Cost posted to Finance on completion |
| `production_audit` | Action log, capped at the newest 200 rows |
| `production_events` | Outbox for Inventory / Finance / Sales events |

To set up a fresh project: run `supabase/schema.sql` in the Supabase SQL
editor, then start the server. The schema grants access to `anon` and creates
the indexes the queries rely on.

`production_audit.user` is stored in a column named `actor`, because `user` is
reserved in Postgres. The server maps between the two.

---

## Order lifecycle

```
material_pending  ──(materials fully issued)──►  ready
       │                                          │
       │                                          ▼
       │                                    in_progress
       │                                          │
       ▼                                          ▼
   cancelled ◄────(cancel with reason)──────  completed
```

| Status | Meaning |
|---|---|
| `material_pending` | Order created; materials not yet fully issued |
| `ready` | All requested materials issued; the order can start |
| `in_progress` | Running through the four stages |
| `completed` | All stages done, output recorded, FG + cost posting created |
| `cancelled` | Cancelled with a reason; stays visible in the list |

Cancelled orders are never hidden. The reason is stored in `notes` and shown
under the status badge.

### The four stages

```
Disinfection  →  Purifying  →  Bottling  →  Packaging
```

Progress is `completed stages / 4`, rounded to a percentage. Orders created
under the old stage names are renamed positionally on startup, and that
migration is fail-safe: if it fails it logs and retries next boot rather than
blocking startup.

### The full chain

1. **Product Master** - define what is produced.
2. **Bill of Materials** - define the recipe and material costs.
3. **Production Planning** - schedule a run; convert it to an order.
4. **Production Order** - the work itself: start, advance, output, complete.
5. **Material Requirement** - explode the BOM into a request for the order.
6. **Material Issuance** - release materials; the order flips to `ready`.
7. **Production Monitoring** - stage progress, timeline, machine status.
8. **Finished Goods** - output recorded, usually automatically on completion.
9. **Production Costing** - standard cost per product, period costing.
10. **Production Reports** - summary, material usage, cost, finished goods.

Completing an order writes three things in one transaction:

- a `production_finished_goods` row (`In Stock`) for the produced quantity,
- a `production_cost_postings` row (`units x unit cost`, status `Pending`),
- audit entries, plus outbox events for Inventory and Finance.

### Costing formula

```
unit cost = sum(bom qty_per_unit x std_unit_cost)
          + labor_per_unit
          + machine_per_unit
```

Material cost is derived from the active BOM every time; labor and machine
rates come from `production_cost_rates`. Nothing is stored as a hardcoded
total.

Current demo values:

| Product | Material | Labor | Machine | Total |
|---|---|---|---|---|
| 5-gal Refill | 4.20 | 1.50 | 1.10 | **6.80** |
| 50 ml Bottled | 2.10 | 0.80 | 0.60 | **3.50** |
| 1 L Bottled | 3.00 | 1.00 | 0.90 | **4.90** |

On completion the posting is `produced units x total unit cost`.

### Outbox events

The module writes to `production_events` instead of calling other modules
directly. Each row has `source_module: 'production'`, a status of `pending`, and
an event type:

| Event | Target module | Fired when |
|---|---|---|
| `production.material_requested` | inventory | A material request is generated |
| `production.material_issued` | inventory | Materials are issued |
| `production.finished_goods_recorded` | inventory | FG is recorded (auto or manual) |
| `production.order_completed` | finance | An order completes, with cost totals |
| `production.order_cancelled` | sales | An order is cancelled, with the reason |

---

## API reference

37 endpoints, all JSON, all under `/api`. Errors return a JSON `{ "error": ... }`
with a 4xx/5xx status.

**System**

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/health` | Storage mode, key kind, project URL, server time |
| GET | `/api/bootstrap` | Everything the UI needs in one call |
| GET | `/api/dashboard` | KPI payload |

**Products and BOM**

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/products` | List products |
| POST | `/api/products` | Create a product |
| PATCH | `/api/products/:id` | Update a product |
| GET | `/api/boms` | List bills of materials |
| POST | `/api/boms` | Create a BOM |

**Orders**

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/orders` | List orders |
| POST | `/api/orders` | Create an order |
| GET | `/api/orders/:id/timeline` | Stage timeline with timestamps |
| POST | `/api/orders/:id/start` | Start the first stage |
| POST | `/api/orders/:id/advance` | Complete the current stage, start the next |
| POST | `/api/orders/:id/output` | Record good/rejected quantities |
| POST | `/api/orders/:id/complete` | Complete, create FG and cost posting |
| POST | `/api/orders/:id/cancel` | Cancel with a reason |
| POST | `/api/orders/:id/material-request` | Generate the material request |

**Planning**

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/plans` | List plans |
| POST | `/api/plans` | Create a plan |
| POST | `/api/plans/:id/convert` | Convert a plan into a production order |

**Material requirement and issuance**

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/material-requests` | List requests |
| POST | `/api/material-requests` | Generate a request for an order |
| POST | `/api/material-requests/:id/approve` | Approve |
| POST | `/api/material-requests/:id/reject` | Reject |
| GET | `/api/issuances` | List issuances |
| POST | `/api/issuances` | Issue materials against an approved request |

**Machines and finished goods**

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/machines` | List machines |
| PATCH | `/api/machines/:id` | Change status (Online / Under Maintenance) |
| GET | `/api/finished-goods` | List finished goods |
| POST | `/api/finished-goods` | Record FG manually |

**Costing and reports**

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/costing/standard` | Standard cost per active product |
| POST | `/api/costing/calculate` | Period cost summary |
| GET | `/api/reports/summary` | Orders completed, units, rejected, yield |
| GET | `/api/reports/material-usage` | Usage aggregated by material |
| GET | `/api/reports/cost` | Cost report for a period |
| GET | `/api/reports/finished-goods` | Finished goods report |

**Audit and events**

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/audit` | Newest 200 audit entries |
| GET | `/api/events` | Outbox rows |

### API guards

Business rules are enforced on the server, not in the UI:

- An order cannot start until materials are fully issued.
- A stage cannot advance if no stage is running.
- Issuance requires an approved request.
- A completed order cannot be cancelled.
- Output cannot exceed the order quantity.

### Health check

`GET /api/health` is the fastest way to diagnose a setup problem:

```json
{
  "ok": true,
  "storage": "supabase",
  "key": "anon",
  "project": "https://<project-ref>.supabase.co",
  "time": "2026-11-12T04:02:23.254Z"
}
```

If `storage` is `json`, the server did not find a URL and key and is running on
the local fallback. If the server exits on startup, the console prints a
checklist: project URL, whether `schema.sql` was run, key type, and grants.

---

## Backups

Every change to the app should be preceded by a snapshot. Backups are zip files
of the source, config, schema, and public assets, written to `backups/`, which
is git-ignored. They are never committed and never deployed.

```bash
npm run backup                  # label: snapshot
npm run backup -- before-ui     # label: before-ui
```

Output: `backups/aquatrack-<label>-<timestamp>.zip`.

The archive contains `server.js`, `seed.js`, `package.json`, `config.json`,
`README.md`, `public/`, and `supabase/`. It does **not** contain `data.json`
because that is runtime state, not source.

Restore by unzipping over the project folder. Do not restore while a server is
running.

---

## Deployment

The live site is deployed on **Render** from a GitHub copy of this project:

| Setting | Value |
|---|---|
| Repository | `LHalcon17/AquaTrackDemo` |
| Branch | `PRODUCTION` |
| Build command | none (no dependencies) |
| Start command | `node server.js` |
| Node version | 18 or newer |

The runtime discovers `PORT` from Render's environment, so no code change is
needed between local and hosted. `config.json` supplies the same Supabase
project, which is why locally created data appears on the live site - they
share one database.

Pushing to that branch triggers a redeploy. Pushing to the original team repo
does not. Check the deployed commit against your local HEAD if a change does
not appear.

Only one instance may run at a time - see "Only one server at a time" above.

---

## Working with the rest of the team

The Production module lives on its own branch and only owns `production_*`
tables. Other modules add their own tables to the same Supabase project and
communicate through the `production_events` outbox.

Rules of thumb:

- Commit only to your own branch.
- Never `git push --force`.
- Never push to `main`.
- Never commit `.env`, `data.json`, `backups/`, or any service key.
- Run `npm run backup` before a significant change.

---

## Feature-to-code map

| # | Screen | API | Front-end |
|---|---|---|---|
| 1 | Product Master | `/api/products` | `renderProducts()` |
| 2 | Bill of Materials | `/api/boms` | `renderBoms()` |
| 3 | Production Order | `/api/orders/*` | `renderOrders()` |
| 4 | Production Planning | `/api/plans/*` | `renderPlans()` |
| 5 | Material Requirement | `/api/material-requests/*` | `renderMaterialRequests()` |
| 6 | Material Issuance | `/api/issuances` | `renderIssuances()` |
| 7 | Production Monitoring | `/api/orders`, `/api/machines` | `renderMonitoring()` |
| 8 | Finished Goods | `/api/finished-goods` | `renderFinishedGoods()` |
| 9 | Production Costing | `/api/costing/*` | `renderCosting()` |
| 10 | Production Reports | `/api/reports/*` | report handlers + CSV export |

All front-end calls go through one helper, `api()`, in
`public/production-api.js`. A single `refresh()` pulls `/api/bootstrap` and
re-renders every panel, so a change on one screen is reflected everywhere after
the next action.

---

## Known limitations

These are deliberate scope boundaries for the demo, not hidden defects. Worth
knowing before a system check:

- **Authentication is not implemented.** The audit actor is a hardcoded
  placeholder (`C. Mendoza`). Login-before-submission was deferred.
- **The audit log keeps the newest 200 entries.** Older entries are dropped.
- **Inventory is not deducted.** The module records issuance and emits an
  event; stock deduction belongs to the Inventory module.
- **"Delivered" status and notifications are not built.** An order ends at
  `completed`; delivery is a Sales concern.
- **The app refreshes on action, not on a timer.** The "Live" badge on
  Monitoring reflects that data is current as of the last action, not that it
  polls continuously.
- **Reports export client-side.** `Export` builds a CSV in the browser from
  data already fetched; no server-side file is generated.
- **Only Production is wired to the backend.** The other modules in the shared
  UI are visual placeholders and show a toast.
- **RLS is off and the anon key is public.** Demo data only.
- **One server at a time.** Multiple instances against the shared database will
  overwrite each other.

---

## License

Internal academic project. Not for redistribution.
