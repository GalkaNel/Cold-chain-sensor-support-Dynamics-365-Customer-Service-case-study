# Security Matrix

**Principle:** privileges split between transactional data (representative) and master data (manager). Delete on cases and events is withheld from representatives — the business case rests on a retained audit trail.

Levels: **BU** business unit · **Org** organisation · **—** none.

| Table | Privilege | Service Agent | Service Manager |
|---|---|---|---|
| Case | Create / Read / Write | BU | BU |
| | Delete | — | BU |
| | Append / Append To | BU | BU |
| Sensor | Read | BU | BU |
| | Create / Write / Delete | — | BU |
| Sensor Event | Read | BU | BU |
| | Create / Write | — | BU |
| | Delete | — | — |
| Account / Contact | Read | BU | BU |
| | Write | — | BU |
| Entitlement | Read | Org | Org |
| | Create / Write | — | BU |
| SLA | Read | Org | Org |
| Queue | Read | BU | BU |
| | Create / Write | — | BU |
| Knowledge Article | Read | Org | Org |
| | Create / Write | — | Org |
| Activity / Task | Create / Read / Write | BU | BU |
| | Delete | — | BU |

Both roles include App Opener privileges.

## Implementation notes

| Point | Why it matters |
|---|---|
| Roles created **inside the solution** | Security roles are solution components; a role created in the admin centre stays in the default solution and is silently missing from the deployment package |
| App Opener privileges required | Without them a role grants table access but the user cannot open the model-driven app at all |
| Sensor Event delete withheld from both roles | Events are raw evidence of what a device reported; corrections belong on the case |
| Entitlement and SLA readable org-wide, editable only by the manager | Representatives must see the commitment that applies; they must not alter commercial terms |

## Verification status

Configured and included in the solution export. **Not exercised with a second user** — trial licensing provided no second licence. Behavioural verification is the first security task of stage two.

## Stage two

| Role | Purpose |
|---|---|
| Field Technician | Read on cases and devices for the assigned site; write on activities only |
| Portal Customer (external) | Own cases and own devices only; identity derived from the authenticated portal user, never selected. Enforced by table permissions and web roles, not business-unit scoping |
