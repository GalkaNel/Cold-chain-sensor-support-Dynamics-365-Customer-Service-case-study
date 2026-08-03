
# Test Log

Customer Service trial environment. Signals injected by HTTP POST (PowerShell); results verified in the app, in Dataverse and in flow run history.

## Scenarios

| ID | Scenario | Expected | Result |
|---|---|---|---|
| TC-01 | Unregistered serial (`FAKE-999`) | 404, nothing written | Pass |
| TC-02 | Excursion, premium customer (`TS-1001`) | Case created, priority High, entitlement auto-applied, premium SLA, AI rationale present | Pass |
| TC-03 | Repeat signal, active case exists (`TS-2001`) | No second case, AI step skipped | Pass — no inference cost |
| TC-04 | Excursion, customer without entitlement (`TS-3001`) | Baseline SLA applied | Pass |
| TC-05 | Queue assignment from intake flow (`TS-3002`) | Case in Critical Response | Pass |
| TC-06 | SLA differentiation, Normal priority | Premium 1 h / 4 h · Standard 2 h / 8 h | Pass |
| TC-07 | High priority, premium contract | 30 min first response | Pass |
| TC-08 | First response recorded | First-response timer stops, resolution continues | Pass |
| TC-09 | Unanswered case past window | Marked escalated, queue moved, task raised | Pass (after DEF-02, DEF-03) |
| TC-10 | Manual `Add to Queue` | Case appears in selected queue | Pass |
| TC-11 | OOTB routing rule set (`Save & Route`) | Case routed by priority | **Fail — environment** |
| TC-12 | Seeder re-run | No duplicates | Pass |
| TC-13 | Seeder dry run (`take(...,1)`) | One record processed | Pass |
| TC-14 | Event record naming | Readable identifier on every event | Pass (after DEF-01) |

## Defects

| ID | Symptom | Root cause | Fix |
|---|---|---|---|
| DEF-01 | Events created with empty name | Explicit `null` is written as a value and suppresses autonumber | Name generated in the flow |
| DEF-02 | Escalation failed, empty `recordId` | Loop iterated the wrong `List rows` output — two actions expose an identically named `value` token | Loop input set by explicit expression |
| DEF-03 | Escalated cases landed in a system queue | Filter on the queue lookup was empty → arbitrary queue returned and bound | Filter restored |
| DEF-04 | No cases created, run reported success | Condition operator inverted relative to branch placement | Operator corrected, condition renamed to state its meaning |
| DEF-05 | Cases created with default priority | Priority not populated in the create action | Priority expression added, including the override |
| DEF-06 | Copied flow would not save | Copying retains parameters of the previous table | Residual `item/...` parameters cleared |
| DEF-07 | Import wizard rejected all rows with lookups | Lookups require a record identifier; text is not resolved, alternate key does not change this | Replaced by a flow that resolves the lookup and binds the identifier |

**In four of seven defects the run reported success.** Verification against expected data, not run status, is what surfaced them; raw action inputs and outputs were the diagnostic tool each time.

## Environment constraints

| Constraint | Position taken |
|---|---|
| Routing rule set produced no queue items | Queue assignment moved into the intake flow — the intended design for an automated channel; manual assignment verified for the human path |
| No second licensed user | Roles documented as configured, not verified |
| No Exchange mailbox | Email channel out of scope; cost of removing the constraint is known |
| Sample data not bulk-deletable in trial | Excluded from the demonstration path |
| Platform API timeouts near trial expiry | Retry after 10–15 min; unmanaged export taken after every block |

## Coverage

Covered — automated channel end to end (intake, validation, deduplication, AI assessment, case creation, prioritisation, routing, SLA, escalation); human path (manual queue assignment, first response, knowledge search); data utilities (idempotency, dry run).

Not covered — multi-user security behaviour, email and portal channels, load and performance, correlation of concurrent events from one site (stage two).
