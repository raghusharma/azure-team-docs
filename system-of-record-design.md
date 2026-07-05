> Published from CloudKeeper's Azure Commit working docs on 2026-07-05. Auto-generated copy — don't edit this file; changes are overwritten on the next publish. Questions / change requests → Raghu Sharma.

# Azure Commit — recommendation system-of-record design

The **upstream model** that Azure Commit summary table schema explicitly defers
("the upstream model is still TBD"). The summary table is the frontend-facing *terminal
sink* (realized economics per resource per usage-date). This doc designs the table that
sits **before** it: the system-of-record for *what we recommended and what happened to it*,
which is what the 25% chargeback is billed against.

Scoped **Commit-first** (Azure Commit is current priority). Azure Tuner reuses the
same spine later — see Tuner reuse (shelved).

## Where it sits (the five boxes)

```
Generate → RECORD → Detect(purchase) → Attribute → Display
 skills    (this    reservation API    realized    summary table
           doc)                        savings     (already designed)
```

- **Generate** — the war-azure RI/SP skills. A scanner. Upserts into Record.
- **Record** — *this doc.* System-of-record: two tables — `recommendation` (mutable target) + `commitment_ledger` (immutable holdings).
- **Detect** — for Commit, "implemented" = *reservation purchased* (billing/reservation API),
  **not** infra-change detection. That (Box 3, `resourcechanges` diffing) is Tuner-only.
- **Attribute** — realized savings from `date_implemented`, computed by the pipeline.
- **Display** — Azure Commit summary table schema. Reads realized savings. Already built.

The customer **Savings Delivered** page shows *realized savings only* → reads the summary
table → **already unblocked, independent of this doc.** This record is internal.

## Pipeline map (the four builds)

Four pipelines across two tracks (advice vs holdings), mapping to the five boxes above:

| # | Pipeline | Reads | Writes | Track | Status |
|---|---|---|---|---|---|
| 1 | **ingest loader** | recommendation run files (S3 `commit/`) | `recommendation`, `recommendation_run_log`, `ingest_run` | advice | built |
| 2 | **commitment loader** | CK purchase records + pre-existing-reservation scan | `commitment_ledger` | holdings | to build |
| 3 | **realized-savings loader** | discounted usage (FOCUS) tagged by `lease_id` | summary table (per-resource realized savings) | holdings | to build |
| 4 | **invoice generator** | `commitment_ledger` ⨝ summary (on `lease_id`) | invoice (a report, not a table) | billing | to build |

**"Commitment loader"** (not "reservation loader"): matches `commitment_ledger`, and *commitment
discount* is the umbrella for both Reserved Instances **and** Savings Plans ("reservation" reads
as RI-only).

The customer **Savings Delivered** page is frontend/API over the summary table — downstream of
#3, not one of the four. Invoice = `25% × realized savings` for holdings where
`source = 'ck-purchased'` and today is within `[fee_start, fee_end]`.

### Repository structure (all four in one repo)

One repo (`azure-commit-pipeline`), one shared `sql/schema.sql`, one Snowflake config —
so the four pipelines live together, not in four repos. Target layout: a shared `common`
package (Snowflake config, DB/MERGE helpers, `errors`, `s3`) + one subpackage per pipeline:

```
azure_commit/
  common/        config, db helpers, errors, s3        ← shared by all four
  ingest/        contract, identity, mapping, pipeline  ← #1 (built)
  commitment/    Azure API client → commitment_ledger   ← #2
  savings/       FOCUS export → summary table           ← #3
  invoice/       ledger ⨝ summary → report              ← #4
```

**Not yet done — deliberately deferred.** #1 currently ships as a flat `commit_loader/`
package (half shared infra, half ingest-specific). The refactor into `common/` + `ingest/`
is the **first step of building #2**, not a big-bang now: that's when a second consumer of
`config`/`db` makes the shared seam real, and it avoids disturbing #1 while it goes to
production on EC2. The rename also kills a naming collision — `commit_loader` (package) vs
the **commitment loader** (#2) is the same reservation-vs-recommendation trap, so the ingest
package becomes `azure_commit/ingest`.

## Core principle: target (mutable) vs holdings (immutable)

A reservation recommendation is **not per-resource.** You don't reserve "for a VM" — you
reserve a *quantity* of a normalized SKU family, in a region, at a billing scope, and Azure
auto-applies the discount to whatever matching usage exists (Instance Size Flexibility). So
the model has two entities:

| Entity | What it is | Mutability | Cardinality |
|---|---|---|---|
| **recommendation** | the recurring *need* / **target** — "this workload warrants N units of coverage" | `recommended_quantity` **re-sized every scan** | one stable row per need |
| **commitment ledger** | a **holding** — an actual reservation/SP that exists (CK-bought *or* pre-existing) | `quantity` **immutable** once bought | many per recommendation |

Everything dynamic falls out of **the delta between target and the sum of active holdings:**

```
coverage = Σ ledger.quantity  WHERE recommendation_id = R  AND status = 'active'
residual = recommendation.recommended_quantity − coverage
  residual > 0  → under-covered  → top-up / next tranche
  residual < 0  → over-committed → wastage (confirm via summary UNUSED_HRS)
  residual = 0  → covered
```

This one relationship handles renewals, laddered partial buys, post-purchase usage drift,
and pre-existing reservations — see Why one structure covers all the dynamics.

### FOCUS attribution: unused-commitment rows carry a null subscription

In the FOCUS export, unused reservation / savings-plan charges (and rounding adjustments) have
`SubAccountId` (subscription) and `ResourceId` set to **null/empty** — by design in the enhanced
export experience, null means "no consuming subscription," not missing data. Attribute these to the
commitment (`CommitmentDiscountId`) as **wastage** — the source for summary `UNUSED_HRS` and the
`residual < 0` over-committed signal above — never to a subscription, and **never coalesce null to
the empty GUID `00000000-…`** (that invents a phantom subscription and misallocates the waste).

### Deterministic identity (the load-bearing decision)

The consolidated CSV is the **union of many scans** — the same need appears every run. A CSV
row is an *observation* with no identity; the DB collapses N observations into one entity
via a deterministic key derived from **what makes a need the same need:**

```
recommendation_id = hash(tenant_id, scope_id, region, service, normalized_sku_family)
```

Each scan `MERGE`-upserts on this key. **`region` is in the identity** — reservations are
region-bound (a West-Europe RI does not cover East-US usage). **`commitment_type` and
`term` are NOT in the identity** — they're the *recommended action* (RI-vs-SP, 1yr-vs-3yr),
decision parameters that can change between tranches/renewals without changing the
underlying need. They live on the holding (what was actually bought).

## Cost basis: customer-effective, NOT retail

Savings are billed at 25% of **savings *delivered***, so the baseline must be the customer's
**actual negotiated rate** (MACC/MCA/EA), sourced from their **FOCUS export**
(`EffectiveCost`/`ContractedCost`) — *not* the Retail Prices API. Billing 25% of
`(retail_OD − reserved)` when the customer already pays ~20% sub-retail (e.g. Eptura)
would overstate savings and overcharge. The Retail Prices API is for reservation *list
pricing* and reference only.

## Status & coverage reconciliation

The recommendation has no linear "purchased" lifecycle — it is **continuously reconciled**
against the live ledger each scan:

- **recommendation.status** = advice state, from three independent sources — `open` (residual > 0)
  · `covered` (residual ≈ 0, a **ledger** fact) · `withdrawn` (dropped from the generator feed) ·
  `dismissed` (an **operator** hid it). Only `covered` is ledger-driven. None touch billing.
- **ledger.status** = holding state: `active` · `expired` · `exchanged` (tranche traded to an SP).

"Already purchased, now what?" is a non-question: the scan recomputes `coverage` and
`residual`; top-ups and wastage are the sign of the residual. There is **no `reverted`
state for Commit** — you can't un-buy a reservation (Azure refund-capped). The MVP
**manual approval gate** sits between a residual-open recommendation and a new ledger
purchase (see Azure Commit capability ladder).

### Withdrawn: feed-absence, not ledger-driven

The generator sends the **whole live set** each run (contract `is_complete`). A recommendation
present last run and **gone this run** — for a scope inside `subscriptions_covered` — means the
engine stopped advising it (customer self-bought, or the workload shrank). It resolves to
`withdrawn`, **not** `covered`: `covered` asserts *we* hold coverage (a ledger fact), which
feed-absence can't tell us. Withdrawal changes advice only — a `withdrawn` recommendation with an
active CK holding **keeps billing**. Never inferred from a `status:failure` or `is_complete:false`
run. The scope-type split (shared vs subscription) lives in the loader steps below.

## Schema (v0)

**Warehouse: Snowflake** (not a separate OLTP store — over-engineering at this write
volume). `MERGE`-upsert; `details` as `VARIANT`; Snowflake-native types matching
Azure Commit summary table schema. Two **core** tables (recommendation + ledger);
the loader adds two small **support** tables — `recommendation_run_log` (history) and
`ingest_run` (idempotency), below.

### Table 1 — `recommendation` (the target)

```sql
recommendation_id            VARCHAR  PRIMARY KEY,  -- deterministic; see identity above
product                      VARCHAR,               -- 'commit'  (later: 'tuner')
tenant_id                    VARCHAR,
scope_type                   VARCHAR,               -- 'shared' | 'subscription' | 'resource_group'
scope_id                     VARCHAR,
region                       VARCHAR,               -- identity; reservations are region-bound
service                      VARCHAR,               -- 'compute' | 'sql' | ...
normalized_sku_family        VARCHAR,               -- identity; ISF family/unit
title                        VARCHAR,

status                       VARCHAR,               -- 'open' | 'covered' | 'withdrawn' | 'dismissed'
recommended_quantity         NUMBER(18,2),          -- MUTABLE target (normalized units); re-sized each scan
recommended_commitment_type  VARCHAR,               -- advice param: 'reserved_instance' | 'savings_plan'
recommended_term             VARCHAR,               -- advice param: 'P1Y' | 'P3Y'

effective_od_monthly_cost    FLOAT,                 -- customer-effective baseline (FOCUS), NOT retail
projected_commit_monthly_cost FLOAT,
estimated_monthly_saving     FLOAT,                 -- projection at current sizing
currency                     VARCHAR,               -- governs the 3 cost columns; per-rec (one run can mix GBP+USD) — aggregations must GROUP BY it

date_first_recommended       TIMESTAMP_NTZ(9),
first_seen_scan              VARCHAR,
last_seen_scan               VARCHAR,
created_at                   TIMESTAMP_NTZ(9),
updated_at                   TIMESTAMP_NTZ(9),
details                      VARIANT                -- evidence: contributing resource_ids + contributions; service extras
```

`coverage` and `residual_quantity` are **derived** (a view over the ledger), not stored.

### Table 2 — `commitment_ledger` (the holdings)

```sql
commitment_id            VARCHAR  PRIMARY KEY,   -- OUR internal id; one row per reservation/SP (or tranche)

-- Azure ids: a reservation has two (order + reservation); a savings plan has its own.
reservation_order_id     VARCHAR,                -- Azure reservationOrderId (null for savings plans)
reservation_id           VARCHAR,                -- Azure reservationId (or savingsPlanId)
lease_id                 VARCHAR,                -- the JOIN key to summary table (which of the above — TBD vs a real FOCUS export)

recommendation_id        VARCHAR,                -- FK (nullable: a pre-existing holding may match no rec)
source                   VARCHAR,                -- 'ck-purchased' | 'customer-pre-existing'

commitment_type          VARCHAR,                -- 'reserved_instance' | 'savings_plan'
service                  VARCHAR,                -- 'compute' | 'sql' | 'appservice' (mapped from reservedResourceType)
normalized_sku_family    VARCHAR,
term                     VARCHAR,                -- 'P1Y' | 'P3Y'
quantity                 NUMBER(18,2),           -- IMMUTABLE units bought
region                   VARCHAR,
billing_plan             VARCHAR,                -- 'Monthly' | 'Upfront'

-- WHO PAYS (billing plane) — reservation.billingScopeId
billing_scope_id         VARCHAR,                -- raw path (a subscription in real EA data)
billing_scope_type       VARCHAR,                -- 'subscription' | 'billing_profile' | 'billing_account'
billing_account_id       VARCHAR,                -- resolved: EA enrollment / MCA billing account; the customer/invoice key (nullable — from cost data)

-- WHERE THE DISCOUNT APPLIES (benefit plane) — appliedScopeType/appliedScopes
applied_scope_type       VARCHAR,                -- 'shared' | 'subscription' | 'management_group' | 'resource_group'
applied_scope_id         VARCHAR,                -- null for 'shared'; sub/RG/MG id otherwise

status                   VARCHAR,                -- 'active' | 'expired' | 'exchanged'
date_purchased           TIMESTAMP_NTZ(9),       -- benefit starts here
term_end_date            DATE,
amortized_upfront_cost   FLOAT,

billable                 BOOLEAN,                -- CK-attributable? (see below)
billable_reason          VARCHAR,                -- 'ck-purchased' | 'pre-existing-customer-ri' | 'customer-self-purchased'
fee_start_date           DATE,                   -- 12-mo fee window PER HOLDING (SOW §8.4)
fee_end_date             DATE,
realized_monthly_saving  FLOAT,                  -- from summary table via lease_id

created_at               TIMESTAMP_NTZ(9),
updated_at               TIMESTAMP_NTZ(9),
details                  VARIANT                 -- raw reservation/savings-plan payload
```

**Validated against a real Azure reservation (Itron tenant, 2026-07-02).** What the live data
forced into the schema:
- A reservation carries **no tenant/customer** — so there is no `tenant_id` here. The customer
  is *resolved* (`billing_account_id`) from the cost/FOCUS side, not read off the reservation.
- **Two scopes, not one.** *Who pays* (`billing_scope_id`, a **subscription** even for
  Shared-scope EA reservations) is separate from *where the discount lands*
  (`applied_scope_type`). The old single `scope_*` conflated them.
- **Two Azure ids** (`reservationOrderId` + `reservationId`); which one the FOCUS export joins on
  is still TBD — hence both are stored and `lease_id` names the chosen one.
- **Savings plans are a different Azure object** (`Microsoft.BillingBenefits/savingsPlans`, not
  `Microsoft.Capacity`), so the commitment loader has two source APIs — reinforcing the name.

### `billable` = attribution, not realization

`billable` answers *"did CK deliver this saving?"* — a fact realization alone can't tell
you. A pre-existing customer RI delivers realized savings but is **not** CK-attributable →
`billable = false`, reason captured. Charged-now is the derived conjunction:

```
chargeable_now = (source = 'ck-purchased')        -- attribution (stored)
             AND (today BETWEEN fee_start AND fee_end)   -- within 12-mo window
             AND (realized_monthly_saving > 0)           -- delivered (from summary)
```

Pre-existing holdings are recorded **anyway** — they cover usage, so sizing must subtract
them: `recommended_quantity` is sized against usage *not already covered by any active
holding* (CK's or the customer's). The ledger feeds both billing and sizing.

### Table 3 — `recommendation_run_log` (append-only history)

`MERGE` overwrites `recommended_quantity` in place, so the target's time-series would be lost.
One immutable row per `(recommendation_id, run_id)` preserves what the engine reported each run —
the audit trail for billing disputes ("what did we advise, and when").

```sql
recommendation_id         VARCHAR,
run_id                    VARCHAR,
scan_seen_at              TIMESTAMP_NTZ(9),
recommended_quantity      NUMBER(18,2),        -- min_wastage, as loaded
estimated_monthly_saving  FLOAT,
status_after              VARCHAR,             -- status this run resolved to
details                   VARIANT,             -- full sizings[] as received
PRIMARY KEY (recommendation_id, run_id)
```

### Table 4 — `ingest_run` (idempotency ledger)

One row per processed file, keyed on `run_id` (contract guarantees uniqueness). The loader skips a
`run_id` already present; upsert + withdrawal + run-log + this row commit in **one transaction**, so
a crash mid-load rolls back cleanly.

```sql
run_id          VARCHAR  PRIMARY KEY,
tenant_id       VARCHAR,
service         VARCHAR,
generated_at    TIMESTAMP_NTZ(9),
count_reported  NUMBER,                        -- header count; must equal recommendations.length
count_loaded    NUMBER,
is_complete     BOOLEAN,
processed_at    TIMESTAMP_NTZ(9)
```

## Why one structure covers all the dynamics

One recommendation → many ledger rows is the same mechanism for four behaviors:

- **Renewals** — 1yr RI expires, workload persists → recommendation re-opens (`residual > 0`);
  a **new** ledger row for the re-purchase. Old row stays `expired` → full history kept.
- **Laddering / partial fills** — buy the target as time-spaced tranches (3 + 3 + 4) to keep
  blast radius small; each tranche is its own ledger row; `coverage` sums them; `residual`
  shrinks as you fill. A partial buy simply leaves the recommendation `open`.
- **Post-purchase usage drift** — instance stopped → `recommended_quantity` re-sizes down;
  holdings stay fixed (you prepaid) → `residual < 0` surfaces the **wastage**; the small
  wasteful tranche can be exchanged to an SP (`status = 'exchanged'`, successor row).
- **Pre-existing reservations** — recorded as holdings with `source = 'customer-pre-existing'`,
  `billable = false`; subtracted from sizing so we never recommend buying existing coverage.

## Flow into the existing summary table

`lease_id` is the **Azure reservation identifier** (exact field — reservationId vs
reservationOrderId — to confirm). `null` until purchased; set on the holding once bought.
Each per-resource row in Azure Commit summary table schema is tagged with the
`LEASE_ID` that provided its benefit. So:

```
1 recommendation → 1 holding (lease_id) → many per-resource summary rows
```

Join `commitment_ledger.lease_id = summary.LEASE_ID` to roll realized per-resource savings
back up to the holding (and thence the recommendation) being billed.

## Loader: JSON → recommendation (Raghu's first build)

Reads one run file (one `(tenant, service)`) from `s3://…/commit/`, per the
[output contract](output-contract.md). Per file, in **one transaction**:

1. **Validate / guard** — reject unknown `version`; skip `status:failure`; **reject if `count` ≠
   `recommendations.length`** (truncated/corrupt file → false withdrawals); **reject if any rec's
   `tenant_id`/`service` ≠ the header** (one file = one `(tenant, service)`; a mismatch would false-
   withdraw the header's whole set). Skip if `run_id` is already in `ingest_run` **for the same
   `(tenant, service)`** (idempotent); **reject** a `run_id` reused for a *different* one (collision —
   skipping it as a "duplicate" would silently drop the file's data).
2. **Identity** — `recommendation_id = sha256` of the five identity fields, fixed order,
   pipe-separated, each `trim`+`lower`.
3. **Upsert** — `MERGE` into `recommendation`; the `min_wastage` sizing fills the scalar columns,
   full `sizings[]` → `details`. Append one row to `recommendation_run_log`.
4. **Withdraw** — recommendations `open` in the DB but **absent this run** → `withdrawn`. **Skip
   withdrawal entirely if `is_complete:false`.** Scope-type split:
   All v1 recs are `shared` scope (App Service included, as of 2026-07-03), keyed by the **billing
   scope** (enrollment ID for EA, billing profile for MCA). Withdrawal is **bounded to the billing
   scopes present in the file** — match only open shared recs whose `scope_id` appears in this run.
   (The `subscription`-scope path — `scope_id ∈ subscriptions_covered` — stays in the code for future
   RG/subscription-scoped recs; v1 emits none.)
   > [!note] Why the shared bound is safe — and its one gap.
   > A shared rec's `scope_id` **is** its billing scope, and both current customers span **multiple**
   > (Itron: `69070706` + `101021785400001`; Eptura likewise). Matching *all* shared recs
   > would wrongly withdraw a scope a *partial* run never scanned (some subs failed, `is_complete`
   > still true for the rest). The `min_wastage` **send-0-never-omit** rule closes this: a scanned
   > scope always emits at least a zero rec, so a `scope_id` absent from the file means that scope
   > wasn't scanned — bounding to present scopes can't false-withdraw it.
   > **Gap:** a scope scanned to *genuinely nothing* (every resource gone, so no rec at all) also
   > drops out, leaving its old recs stale-`open` rather than `withdrawn`. That's the safe direction
   > (under-withdraw, never over-withdraw). A run-declared **billing-scopes-covered** list (the shared
   > analogue of `subscriptions_covered`) would close it precisely; deferred until the contract adds it.
5. Record the file in `ingest_run`; commit.

New `open` recommendations surface to the operator queue.

Ledger population (CK purchase records + pre-existing-reservation scan) is the **reservations
loader** — a sibling MVP build on the holdings path, not this loader.

**Code:** implemented in the `azure-commit-loader` repo (kept out of the vault — code isn't
vault content). Its README links back to this design and the output contract.

## Tuner reuse (shelved)

Same spine (identity, target/holding split, chargeback), different engine. When Tuner lands:
add `product = 'tuner'`; holdings become infra-change states; **Detect becomes infra-change
diffing** (Box 3, `resourcechanges` — note the ~14-day retention gotcha); add a `reverted` state (Tuner changes *are* revertible). **Do not design this now.**

## Decisions locked (2026-06-15)

- Identity = `(tenant, scope, region, service, normalized_sku_family)`; **region in**,
  **term + commitment_type out** (they're action params on the holding).
- Two tables: recommendation (mutable target) + commitment ledger (immutable holdings).
- `billable` = attribution flag (stored) × within-window × realized (derived).
- Cost basis = customer-effective (FOCUS), not retail.

## Decisions locked (2026-07-01, ingest loader)

- **Sizings** — `min_wastage` fills the scalar columns (drives residual); `advance` → `details`
  (display only; promote to real columns only if the frontend filters/sorts on it).
- **`withdrawn`** — fourth `recommendation.status` for feed-absence; advice-only, never billing;
  the row is kept, not deleted. Distinct from ledger-driven `covered` and operator `dismissed`.
- **Append-only `recommendation_run_log`** — preserves the target time-series `MERGE` overwrites.
- **Identity hash** — `sha256`, five fields, fixed order, pipe-separated, each `trim`+`lower`; hex PK.
- **Idempotency** — `ingest_run` ledger keyed on `run_id`; one transaction per file; **reject the
  file if `count` ≠ list length**.
- **Lifecycle model** = durable row + status transitions — *not* Tuner's snapshot+diff, which is
  wrong for partial-fill, non-revertible, ledger-backed needs (Tuner is per-resource + revertible).

**Still open:**
- `OD_COST` semantics in the summary table — must be customer-effective for chargeback. Raised in Azure Commit summary table schema.
- Whether a renewal opens a **new** 12-month fee window (commercial — re-confirm against SOW §8.4).
- Exchange lineage (`replaced_by` pointer) — shelved; status + successor row suffices for v0.
