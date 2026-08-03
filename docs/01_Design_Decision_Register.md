# Design Decision & Fit-Gap Register

**Project:** ColdChain Sensor Support — Dynamics 365 Customer Service demonstration solution
**Author:** Galina Nelyubova · **Date:** July 2026 · **Stage:** 1 of 2 (core delivered)

## Summary (30 seconds)

**What the solution does.** Sensor signal → device validated → AI assesses severity and writes its reasoning into the case → case created for the right customer → priority set by event type and contract tier → routed to the matching queue → SLA timer per contract → escalation if the response window is missed.

**Functional areas covered:** case management · queues and routing · SLAs and KPIs · entitlements (contract tiers) · knowledge management · business process flow · activities and timeline · security roles · data migration · ALM.

**Skills demonstrated:** requirements framing around a business problem · data modelling in Dataverse · process automation · system integration (inbound webhook) · applied AI with cost and governance reasoning · security design · fit-gap analysis and documented design decisions · test evidence · solution/environment lifecycle · licensing and TCO awareness.

**Tools and components used:** Dynamics 365 Customer Service (Enterprise trial) · Dataverse (custom tables, alternate keys, cascade behaviour, choice sets, autonumber, native statecode/statuscode) · model-driven app · Power Automate (HTTP-triggered intake flow, scheduled escalation flow, two idempotent data-seeding flows, OData filters, dedupe patterns, dry-run technique) · AI Builder prompt (GPT-4.1 mini, structured JSON output) · Customer Service configuration (SLAs, SLA KPIs, entitlements, queues, knowledge articles, OOTB business process flow) · security roles · solutions and unmanaged exports · PowerShell for webhook simulation.

**Five decisions worth reading first:**

1. **Custom Sensor table instead of Field Service Customer Assets** — licensing and functional surface not justified by a monitoring-only scenario (2.1).
2. **"LLM for judgment, flow for guarantees"** — the model assesses and explains; the temperature-excursion priority rule is enforced deterministically in the flow (6.1).
3. **AI inference placed after the duplicate check** — a chattering device cannot burn inference credits with no business value (5.5).
4. **Conversational analytics agent deliberately rejected** — aggregation is where LLMs hallucinate; deterministic questions belong in views and BI (6.3).
5. **Solutions carry schema, not records** — queues, SLAs, entitlements and knowledge articles were restored from documentation, which is what Configuration Migration Tool exists for in production (8.4).

## How to read this document

Every row is a decision that was actually made during build, with the alternatives that were considered and rejected. Where a capability was deliberately left out, it is recorded as a **decision**, not as an omission. Constraints imposed by the trial environment are marked as such, with the production-grade alternative named.

Legend — **Status:** `Built` (implemented and tested) · `Deferred` (planned, Stage 2) · `Rejected` (deliberately not done) · `Constraint` (environment limitation, workaround documented).

---

## 1. Solution scope and business framing

| # | Decision | Alternatives considered | Rationale | Trade-off | Status |
|---|---|---|---|---|---|
| 1.1 | Value proposition framed around **compliance evidence and spoilage cost**, not "faster ticket handling" | Generic service-desk framing | Cold-chain clients in NZ operate under MPI/HACCP; the buyer pays for provable temperature continuity and for avoided stock loss, not for ticket throughput | Requires more domain reasoning in design; harder to demo with vanity metrics | Built |
| 1.2 | Contract tiers named as **service commitments** ("Standard Monitoring", "Continuous Cold-Chain Assurance") | Bronze / Silver / Gold | Tier names should reflect the obligation being sold; generic metal tiers read as demo filler and carry no meaning to the client | Longer labels in UI | Built |
| 1.3 | KPIs presented as **the system's ability to measure**, never as achieved business outcomes | Quoting invented improvement percentages | Fabricated results are unverifiable and damage credibility in review | Less impressive headline numbers | Built |
| 1.4 | Two-stage delivery: core Customer Service first, portal + AI agents second | Single large build | Environment window was 15 days; core end-to-end working beats a broad half-built scope | Portal and agents absent from Stage 1 evidence | Built |

## 2. Data model

| # | Decision | Alternatives considered | Rationale | Trade-off | Status |
|---|---|---|---|---|---|
| 2.1 | **Custom `Sensor` table** instead of OOTB Customer Asset / Field Service | Field Service module with Customer Assets and Connected Field Service | Monitoring-only scenario: Field Service brings licensing cost and a large functional surface (work orders, scheduling, resources) that the business case does not need | Loses OOTB asset hierarchy and IoT alert plumbing; would be revisited if field dispatch entered scope | Built |
| 2.2 | **Exception-only ingestion**: only out-of-range / fault events are persisted as `Sensor Event` | Persisting all telemetry readings in Dataverse | Continuous telemetry at device scale is expensive in Dataverse storage and adds no case-management value; the Event Type choice set therefore deliberately excludes "Normal" | Trend analysis requires the upstream store; in production full time-series belongs in Azure (IoT Hub / Data Explorer) with Dataverse holding exceptions only | Built |
| 2.3 | Sensor primary column = **human-readable name**; `Serial No` a separate Business Required column with an **alternate key** | Serial number as the primary name column | "Names for people, keys for machines": lookups and views stay readable, while integration resolves devices deterministically by serial | Two fields to maintain | Built |
| 2.4 | Business Required treated as a **contract with every write channel**, not as "important field" | Marking all significant fields required | A required field must be guaranteed at creation time by *every* channel (flow, API, UI). `Temperature` is optional because connectivity and battery events carry no reading; AI fields are optional because they are written after record creation | Slightly weaker UI-level enforcement; where a business rule is needed, form-level Business Rules are the right tool | Built |
| 2.5 | Sensor lifecycle uses **native `statecode` / `statuscode`** (Operational / Faulty under Active, Retired under Inactive) | Custom "Sensor Status" choice column (initially generated, then removed) | Duplicating platform state in a custom column breaks standard views, filters and deactivate behaviour | Status reason values must be maintained per state | Built |
| 2.6 | `Location` kept as a **text field** on Sensor | Dedicated Location table with its own relationships | For the demo scope a text field is sufficient; a Location table only pays off when multiple entities reference sites (routing, reporting, contracts per site) | No site-level aggregation | Deferred |
| 2.7 | Standard tables (Account, Contact, Case, Entitlement, Queue, Knowledge Article) **used as-is** | Custom equivalents | Standard tables carry the entire Customer Service mechanism (SLA, entitlements, routing, timeline); replacing them would break the platform | Must live with sample data and OOTB field sets | Built |
| 2.8 | Cascade behaviour: **Referential + Restrict Delete** on Account→Sensor, Sensor→Sensor Event, Sensor Event→Case | Parental cascade; Remove Link on delete | Restrict protects the evidence chain that the compliance story depends on: a device or event cannot be deleted while dependent records exist. Remove Link would silently orphan records that the model declares mandatory | Deletion requires deliberate cleanup order | Built |

## 3. Case intake and channels

| # | Decision | Alternatives considered | Rationale | Trade-off | Status |
|---|---|---|---|---|---|
| 3.1 | Primary channel is **device telemetry via webhook** (simulated by HTTP POST) | Manual case entry as the primary path | The product's value is that the vendor learns about an excursion before the client notices; a manual-first design contradicts the proposition | Demo requires an explicit note that the signal is simulated | Built |
| 3.2 | **Phone treated as an activity record**, never as an intake channel | "Operator logs the call as a case" flow | Logging an outbound call on the timeline is legitimate practice; building phone intake into an IoT monitoring product signals an outdated process design | No telephony demo | Rejected (as channel) |
| 3.3 | Client self-service channel = **Power Pages portal**, not a Canvas app | Canvas app for site managers | Canvas apps cannot be licensed to external customers; presenting one as a customer channel would be licensing-unrealistic | Portal not built in Stage 1; the trade-off (Canvas vs Pages vs guest access) is documented instead | Deferred |
| 3.4 | **Email-to-case not implemented** | Configuring the email channel | No Exchange mailbox in the trial environment. Recorded as an environment constraint with a named cost to remove it (M365 Business Basic, ~USD 13/month) rather than as a gap | One realistic channel missing from the demo | Constraint |
| 3.5 | Categorisation via **Event Type** on the event record | OOTB Subject tree | A second classifier would duplicate meaning and drift from the automation logic that already branches on event type | Subject field left empty; sample subject values remain in the trial org | Built |

## 4. Prioritisation, SLA and entitlements

| # | Decision | Alternatives considered | Rationale | Trade-off | Status |
|---|---|---|---|---|---|
| 4.1 | Priority derived from **contract level + event type + site criticality**, with a hard **domain override**: temperature out of range can never be lower than High | Pure AI-assigned priority; static priority per event type | Business guarantees must not depend on model behaviour. The override is implemented in the flow, deterministically | Slightly less "intelligent" looking demo; deliberate | Built |
| 4.2 | **Entitlement holds the contract level**, linked to Account | Custom tier field on Account; tier on the sensor | One source of truth: the same record drives both SLA selection and priority scoring, which is how commercial obligation actually attaches to a customer | Entitlement records must be activated and set as default per customer | Built |
| 4.3 | **Global default SLA = Standard** (not Premium) | Premium as environment default | A customer without a contract should receive baseline commitments; defaulting to the best terms gives away the product | Requires explicitly setting the default after testing | Built |
| 4.4 | Two SLAs with **deliberately short demo durations** (High 15/30 min; Standard 1/2 h) | Realistic multi-hour thresholds | Timers, warnings and escalation must be observable within a recorded demo | Durations are not production-realistic; stated openly in the demo | Built |
| 4.5 | SLA pause conditions left at **environment default** (On Hold / Waiting for details) | Per-KPI override | Default already reflects the fair-measurement principle: time waiting on the customer should not count against the service team | Not demonstrated explicitly | Built |

## 5. Automation architecture

| # | Decision | Alternatives considered | Rationale | Trade-off | Status |
|---|---|---|---|---|---|
| 5.1 | All asynchronous logic in **cloud flows**; classic background workflows not used | Classic Dataverse workflows | Building new functionality on the legacy engine signals outdated practice; cloud flows are the current platform direction | Legacy workflow maintenance skill demonstrated by documentation only | Rejected |
| 5.2 | One **real-time workflow** reserved for a genuinely synchronous rule: block case resolution without a linked knowledge article | Plugin; cloud flow | Synchronous, no-code validation is the remaining legitimate niche of the classic engine. Tool hierarchy: cloud flow (async) → real-time workflow (sync validation) → plugin (logic beyond declarative) | Not yet built | Deferred |
| 5.3 | Webhook returns **202 Accepted before the expensive work**, and **404 + Terminate** for an unknown serial | Single response at the end of the flow | A device integration must be acknowledged immediately and must not create records for unrecognised senders | Caller does not receive the case number synchronously | Built |
| 5.4 | **Deduplication by natural key**: an active case whose title carries the serial suppresses further case creation | Creating a case per event; correlating by lookup | Prevents case storms from a repeatedly firing device; the natural key is cheap and readable in the demo | Title-based matching is a simplification; a dedicated Case→Sensor lookup is the production form | Built |
| 5.5 | **AI assessment placed after the deduplication check** | Assessing every incoming event | Inference is only needed when a case is actually being created; a chattering device would otherwise consume credits with no business value | Duplicate events carry no AI commentary | Built |
| 5.6 | Queue assignment performed **inside the intake flow** | OOTB Routing Rule Set | The rule set was configured correctly and activated, but `Save & Route` did not create queue items in this trial environment. Flow-based assignment is the target design for an automated channel in any case; manual `Add to Queue` was verified as working for the human path | Rule-set behaviour not demonstrable in the recording | Constraint |
| 5.7 | Queue moves implemented as **delete + create** of the queue item | Updating the queue item's queue field | The `Queue` field on Queue Item is create-only. Native alternative is the bound `AddToQueue` action; delete+create is functionally equivalent and more readable in a demo | Loses queue-item history | Built |
| 5.8 | Escalation runs on a **schedule** (recurrence) and selects overdue cases | SLA failure actions on the SLA item; SLA KPI Instance trigger | The worker logic is identical whichever way it is triggered; a schedule is deterministic, demonstrable on demand and independent of trial-environment behaviour | Threshold is a constant measured from `Created On` rather than from the customer's SLA KPI — the production form would read the contract | Built |
| 5.9 | Autonumber names generated **in the flow** for event records | Passing an explicit `null` to the autonumber column | An explicitly passed `null` is written as an empty value and suppresses autonumber generation; generating the name in the flow is predictable | Numbering is timestamp-based, not sequential | Built |

## 6. AI design and governance

| # | Decision | Alternatives considered | Rationale | Trade-off | Status |
|---|---|---|---|---|---|
| 6.1 | **"LLM for judgment, flow for guarantees"** — the model assesses severity and writes a justification; business rules are enforced in code paths | Letting the model set priority directly | Deterministic obligations (temperature excursions are always High) must be provable, not probabilistic | Model output can disagree with the final priority; the justification explains the assessment, the flow enforces the rule | Built |
| 6.2 | Model returns **structured JSON** (`severity` + `justification`), with the written rationale stored on the event and copied into the case description | Free-text output; severity only | The justification is the actual value: the representative opens a case that already explains why it matters, and the reasoning is retained for audit | Prompt must constrain output format; cost per call ≈ 0.2 Copilot credits (~USD 0.002) | Built |
| 6.3 | **Conversational analytics agent rejected** ("how many sensors, what wears out fastest, who is due for renewal") | Building it as the headline AI feature | Aggregation over records is where LLMs hallucinate, and a demo with a wrong number destroys trust. Such questions are deterministic and belong in views, flows or Power BI — and the built-in model-driven Copilot already answers them | No conversational showpiece | Rejected |
| 6.4 | Document analysis reserved for the **claim scenario** (supplier invoice for spoiled stock inside a real insurance-claim process) | "Agent reads a PDF" as a standalone demo | Document parsing must be justified by the process, not staged for effect | Not built in Stage 1 | Deferred |
| 6.5 | **Human review mandatory** before any AI-assembled evidence pack reaches a customer | Fully autonomous agent responses | Accountability for compliance-grade output stays with a person; autonomy would be technically achievable and commercially indefensible here | Slower than full automation | Deferred (design) |
| 6.6 | AI Builder prompt kept as a **reusable solution component**, invoked from the flow | Inline prompt text in the flow | The prompt is versioned with the solution and can be reused or replaced by a Copilot Studio agent without touching flow logic | — | Built |

## 7. Security

| # | Decision | Alternatives considered | Rationale | Trade-off | Status |
|---|---|---|---|---|---|
| 7.1 | Two roles: **Service Agent** (transactional data) and **Service Manager** (master data + delete) | Single role; four-role model | Separating transactional work from master data (sensors, entitlements, queues, knowledge) is the standard privilege split and is enough to demonstrate the principle at this scope | Four-role model (including a portal-facing role) postponed to Stage 2 | Built |
| 7.2 | Roles created **inside the solution** | Creating them in the admin centre | Security roles are solution components; roles created outside the solution would not travel with the export | — | Built |
| 7.3 | No delete privilege for agents on cases or events | Full CRUD for agents | Deletion breaks the compliance evidence chain that the business case is built on | Cleanup requires a manager | Built |
| 7.4 | Roles **configured but not verified with a second user** | Adding a test user | Trial licensing did not allow a second licensed user. Stated openly rather than implied as tested | Role behaviour is designed, not proven | Constraint |
| 7.5 | Customer identity in any future portal must be **derived from the login**, never chosen by the user | Letting the user pick their company | Foundational authentication principle for external-facing surfaces | — | Deferred (design) |

## 8. ALM, data and delivery

| # | Decision | Alternatives considered | Rationale | Trade-off | Status |
|---|---|---|---|---|---|
| 8.1 | **Segmented solution**: standard tables added as empty references; only genuinely modified components (one form, one column) included | Adding tables with all objects | Keeps the export small and portable and avoids carrying unrelated components between environments | Requires discipline when touching standard components | Built |
| 8.2 | Custom publisher prefix (`gn`) created as the **first action**, before any component | Default publisher | A default publisher prefix is effectively permanent and marks the solution as unmanaged practice; this repeats a known lesson from an earlier project | — | Built |
| 8.3 | **Seeder flows kept out of the product solution** (tooling, not product) | Shipping them with the solution | Utilities do not belong in a deliverable; scaffolding is removed before handover | Must be exported separately for reuse (flows can only be exported inside a solution) | Built |
| 8.4 | Configuration data (queues, SLAs, entitlements, knowledge articles) **restored from documentation**, not from the solution | Assuming the solution carries them | Solutions carry schema and customisations, not records. This is a common and costly misunderstanding | Manual restore step; in production this is Configuration Migration Tool / `pac data` | Built |
| 8.5 | Demo data loaded by **idempotent seeder flows** (dedupe on natural key, dry-run via `take(...,1)`) | Manual entry; import wizard alone | The import wizard does not resolve lookups from text values (GUID required), so a "translator" step is needed; idempotency makes reloads safe and repeatable | Extra build effort for demo data | Built |
| 8.6 | Migration approach documented by volume: manual → import wizard → dataflows → ETL (ADF / third-party) with upsert on alternate keys | Presenting one method as "the" method | Choosing by volume and repeatability is the actual consulting judgement | — | Built |
| 8.7 | Unmanaged solution exported after every completed block | Exporting once at the end | The environment was a 30-day trial that degraded near expiry (API timeouts on save and export); frequent exports were the insurance policy | Slight overhead | Built |
| 8.8 | Microsoft sample data left in place in the trial | Removing it | Bulk deletion of sample data and configuration is not supported in trial environments; sample records were simply excluded from the demo path | Sample knowledge articles and subjects visible in the org | Constraint |

## 9. Licensing and cost awareness

| # | Decision | Alternatives considered | Rationale | Trade-off | Status |
|---|---|---|---|---|---|
| 9.1 | Delivered on **trial + pay-as-you-go**, not on monthly-commit licences | Purchasing Customer Service Enterprise monthly (~USD 126/user/month plus add-ons) | For a demonstration build, PAYG for Copilot credits and Power Pages authenticated users costs roughly NZD 30–50 per active month against ~NZD 270 for the licensed path | Trial banners and 30-day windows; environment cannot be shown publicly | Built |
| 9.2 | Customer Service **Enterprise** identified as the minimum viable SKU if licences were bought | Professional | Professional lacks full entitlement functionality, which the whole contract-tier design depends on | — | Documented |
| 9.3 | Code Apps acknowledged but not built | Building a code app to show breadth | GA since February 2026 and premium-only; it is a pro-developer surface that would blur a functional-consultant positioning | Covered as awareness in documentation only | Rejected |

---

## Open questions carried into Stage 2

1. Whether to buy M365 Business Basic (~USD 13/month) to demonstrate a real email channel and native Teams adaptive cards, or keep the documented constraint.
2. Scope of the compliance report on the portal: generated document versus temperature-log extract.
3. Whether the escalation trigger moves from schedule to SLA failure actions once the mechanism can be verified in a healthy environment.
4. Whether Copilot Studio agent authoring is fully available inside the Customer Service trial, or requires separate pay-as-you-go capacity.

## Known limitations of the Stage 1 evidence

- Security roles are configured, not verified with a second licensed user.
- Routing rule set behaviour could not be demonstrated in the trial; queue assignment is shown via flow and manual add-to-queue.
- SLA and escalation thresholds are demo-short and measured from case creation, not from contract terms.
- Telemetry is simulated by HTTP POST; no physical device or IoT gateway is involved.
- Portal, Copilot Studio agents, renewal opportunity, battery-level prediction, Power BI reporting and the plug-in correlation logic are Stage 2 scope and are not part of this evidence.
