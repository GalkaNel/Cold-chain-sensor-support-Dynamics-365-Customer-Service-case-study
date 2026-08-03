# ALM and Data Migration

Single unmanaged solution `IOT_service`, custom publisher (prefix `gn`) created as the **first action** of the build, before any component existed.

**Segmented:** standard tables added as references without objects; only genuinely modified components included (one Case form, one Case column). Adding tables with all objects inflates the package and creates import conflicts over components nobody touched.

## Solution contents

| Type | Items |
|---|---|
| Custom tables | Sensor, Sensor Event |
| Standard tables (referenced) | Account, Contact, Case, Entitlement, Queue, Knowledge Article |
| Table components | Case: 1 custom column (Originating Event), 1 main form |
| Cloud flows | Sensor Intake, SLA Escalation |
| AI model | Severity assessment prompt |
| Apps | IoT Service Admin (model-driven) + site map |
| Security roles | Service Agent, Service Manager |
| Choice sets | AI Severity |

## What solutions carry — and what they do not

| Carried | **Not** carried |
|---|---|
| Tables, columns, keys, relationships | Records of any kind |
| Forms, views, charts (when added) | Queues |
| Flows, apps, site maps | SLAs, SLA KPIs, SLA items |
| Security roles | Entitlements |
| Choice sets, AI prompts | Knowledge articles |
| Business process flows | Routing rule sets, demo data |

Right column = everything that must be migrated separately. Here: documented and recreated manually, data reloaded by seeder flows. In production: **Configuration Migration Tool** or `pac data export/import`, wrapped in a Package Deployer package so schema and configuration data deploy as one unit.

## Data migration by volume

| Volume | Method | Note |
|---|---|---|
| Tens | Manual / editable grid | Cheapest for reference and config data |
| Hundreds–thousands, one-off | Import wizard, dataflows | Wizard does **not** resolve lookups from text — identifiers required. Dataflows transform and re-run |
| Thousands, repeated, from legacy | ETL (Azure Data Factory, SSIS-based tooling) | Upsert on alternate keys, batching, deduplication, error logs |

**Observed:** the import wizard rejected every row whose lookup was supplied as text (`"Southern Fresh Foods Ltd is not a valid primary id Guid value"`); adding an alternate key did not change it. Working pattern — query the target by natural key, take the returned identifier, bind the lookup with it. That is what dataflows and ETL tools do internally, and the reason they are correct above trivial volumes.

**Idempotency:** seeders match on natural key (email / serial number) before creating, so a re-run creates nothing and can safely restore an environment. Field mapping validated by a dry run limited to one record (`take(...,1)`) before each full load.

## Practices applied

| Practice | Reason |
|---|---|
| Unmanaged export after every completed block | Trial environment degraded near expiry — flow saves and exports returned API timeouts; nothing was ever at risk |
| Utilities kept out of the product solution | Seeder flows are scaffolding, not deliverable. Flows can only be exported inside a solution, so utilities get their own |
| Custom publisher before first component | A default prefix is permanent on everything created under it |

## Environment restore checklist

1. Provision a Customer Service trial environment.
2. Import the unmanaged solution from `solution/`.
3. Recreate configuration records in order: queues → SLA KPIs → SLAs and SLA items → entitlements (activate, then set as default per customer) → knowledge articles.
4. Run seeders: accounts → contacts → devices.
5. Reconnect flow connections, enable both flows, capture the new webhook URL from the intake trigger.
6. Verify with test log scenarios TC-01 … TC-09.

Steps 3 and 4 are what a solution export cannot do — the practical reason this document exists.
