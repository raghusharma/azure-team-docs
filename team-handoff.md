> Published from CloudKeeper's Azure Commit working docs on 2026-07-07. Auto-generated copy — don't edit this file; changes are overwritten on the next publish. Questions / change requests → Raghu Sharma.

# Azure Commit contract — team handoff

What we want: your engine produces the **same recommendations it already produces**, but as a **JSON file per run** instead of the Excel. Same numbers, new shape. The pipeline reads that file, loads it to the DB, and bills off it.

Covers **all three v1 services** — App Service, VM, SQL — plus the **proposed** shape for **Log Analytics / Sentinel tiers** (new section near the end; design pending pipeline-side agreement). App Service is worked in full below (against full_account_recs.xlsx, 2026-06-24); **VM & SQL** share the same file shape and rules, with their differences in the **VM & SQL** section near the end. Nothing here changes *how* you find recommendations — only how you hand them over.

Full spec: [Azure Commit recommendation output contract](output-contract.md). SKU families for VM/SQL: [Azure reservation SKU families (ISF) reference](isf-sku-families-reference.md). This note is the short version.

## The file: one header + one list

One run = one JSON file.

```jsonc
{
  // header — what this run is
  "version": "1.0",
  "run_id": "2026-06-24T10-36-02Z_eptura_appservice",   // unique per run — full UTC timestamp to seconds (multiple runs/day must not collide); filename-safe, no colons
  "tenant_id": "adb48caa-0b14-4256-b178-a5e508c807f5",
  "service": "appservice",                     // 'compute' | 'sql' | 'appservice'
  "subscriptions_covered": [                   // every sub this run scanned (you scan ~79) — list ALL of them, even those with no recs
    "<sub-guid-1>", "<sub-guid-2>", "..."
  ],
  "generated_at": "2026-06-24T10:36:02Z",
  "status": "success",                         // 'success' | 'failure'
  "is_complete": true,                         // true = the WHOLE live set for this service across subscriptions_covered
  "count": 44,                                 // = recommendations.length (illustrative) — one per (billing scope, region, SKU)

  "recommendations": [ /* one item per (billing scope, region, SKU) need; floor + advance on the SAME item — below */ ]
}
```

**v1: App Service ships at `shared` scope, and rolls up plans.** The buyable unit is one `(billing scope, region, SKU)` — an App Service reservation isn't tied to a plan *or* a subscription; at shared scope it floats across **every plan of that SKU across all subs in the billing scope**. So aggregate all those plans into ONE recommendation: `scope_type: "shared"`, `scope_id` = the billing profile ID (MCA) / billing account (EA), the same as VM/SQL (the **VM & SQL** section near the end explains where that ID comes from). Emitting your current **per-plan** rows would make many share one identity and overwrite each other — that's the duplicate-id bug you hit. This collapses the 204 per-plan rows to one rec per SKU per billing scope.

> **Size on merged usage, not the sum of per-plan floors**
> The reservation quantity is the floor of the **merged** usage across all the plans it covers — **not** `Σ` of each plan's floor. Summing over-reserves when plans are staggered (their peaks don't line up); it's only equal when every plan is always-on. This is the one real sizing change for App Service — VM/SQL already size on merged usage.

List **every** scanned sub in `subscriptions_covered` (completeness boundary + audit). A sub that **failed** to scan: leave it *out* of the list, don't silently include it — the pipeline only retires a vanished recommendation if its sub was scanned this run.

## App Service — your Excel columns → the JSON

Group your **Min-Wastage** rows by `(billing scope, Region, SKU, OS)` — across **all** subs in the billing scope — and emit one item per group. (The example below shows one sub's 13 `P0V3` plans for concreteness; in a real run this rec also absorbs the same-SKU plans from the billing scope's other subs.)

| Your column | JSON field | Note |
|---|---|---|
| `Tenant ID` (Summary) | `tenant_id` | as-is |
| (billing scope, from FOCUS/cost data) | `scope_type` = `shared`, `scope_id` = billing profile ID | same as VM/SQL — bare ID, no ARM path |
| `Region` = `UK South` | `region` = `uksouth` | ⚠️ **canonical name**, lowercase no spaces — see below |
| `SKU` + `OS` = `P0V3` / `Windows` | `normalized_sku_family` = `P0V3 Windows` | App Service has no ISF → the SKU **is** the family |
| (service) | `service` = `appservice` | new value, fine |
| `Billing currency` | `currency` = `GBP` | **on each recommendation** (not the header) — the billing scope's currency |
| (which sheet) | `sizings[].strategy` = `min_wastage` / `advance` | **not** a separate row — see below |
| `Reserved Qty`, **merged across the group's plans** | `sizings[].quantity` | floor of merged usage, **not** the sum (see warning above) |
| `Term` = `1-year` (Summary) | `recommended_term` = `P1Y` | `P1Y` / `P3Y` |
| `Current Cost (window)`, summed | `effective_od_monthly_cost` | ÷ to **monthly**, and ⚠️ **customer-effective**, not retail |
| `Reserved Cost (window)`, summed | `sizings[].projected_commit_monthly_cost` | ÷ to monthly, inside the sizing |
| `Plan` (all in the group) | `evidence.contributing_plans[]` | **array** — every plan feeding this need |
| `Subscription` (all in the group) | `evidence.contributing_subscriptions[]` | **array** — the subs that fed it |
| `Current Instances`, `Uptime %`, `Window Inst-Hours` | `evidence{}` | optional, nice to have |

Filled in:

```jsonc
{
  // identity — 5 fields, spell them the same way every run
  "tenant_id": "adb48caa-0b14-4256-b178-a5e508c807f5",
  "scope_type": "shared",
  "scope_id": "XXXX-XXXX-XXX-XXX",             // billing profile ID (bare), from the FOCUS/cost data
  "region": "uksouth",
  "service": "appservice",
  "normalized_sku_family": "P0V3 Windows",

  "title": "Reserve P0V3 (Windows) App Service — UK South",
  "recommended_commitment_type": "reserved_instance",
  "recommended_term": "P1Y",
  "currency": "GBP",                           // per-rec (the billing scope's currency), NOT the header
  "effective_od_monthly_cost": 2373.76,        // monthly, customer-effective, summed across the plans

  // floor (min_wastage) + advance on the same item — not as two rows
  "sizings": [
    { "strategy": "min_wastage",
      "quantity": 18,                          // ILLUSTRATIVE — floor of the MERGED usage; ≤ 19 (the old per-plan sum). recompute on merged usage, don't sum
      "projected_commit_monthly_cost": 1026.83,
      "estimated_monthly_saving": 1346.93 }    // = od − commit, on the monthly basis
    // + an "advance" entry here if the Advance-Saving pass sized this need
  ],

  "evidence": {
    "contributing_plans": [                    // the plans this reservation covers (13 shown; real run spans the billing scope's subs)
      "ukcondeco2291", "ukcondeco1024", "ukcondeco2202", "ukcondeco2190",
      "ukcondeco1022", "ukcondeco2238", "ukcondeco0983", "ukcondeco0945adm",
      "ukcondeco2286adm", "ukcondeco2312adm", "ukcondeco0906", "ukcondeco1016",
      "ukcondeco2191"
    ],
    "contributing_subscriptions": ["<sub-guid-1>", "..."],
    "window_days": 60
  }
}
```

**You do not send an id.** The pipeline builds the id from the 5 identity fields. Your only job there: spell those 5 the same way every run, so the same need lands on the same row next time.

## 4 rules to get right

1. **`is_complete: true` = the whole live set** for this service across the subs in `subscriptions_covered`, not a delta or a page. If a recommendation was in last week's file and is gone this week, the pipeline reads that as **"customer acted on it / it's resolved"** — but only for subs in `subscriptions_covered`. A sub you didn't scan, just leave out of the list (still fine to send `is_complete: true` for the rest). If you can't produce the full set even for the covered subs, send `is_complete: false`.
2. **`status: "failure"` → empty list + a reason.** A crashed run is **not** "everything disappeared." If the engine fails, send `status: "failure"`, `recommendations: []`, and a `reason`. The pipeline skips it. (Otherwise a crash reads as "all savings delivered" → wrong invoice.)
3. **One run = one whole file.** No half-written files. Write to a temp name, rename when done.
4. **Costs are the customer's real rate**, not retail. Your `Pricing Source` today is `retail-discount-estimate` — that overstates savings for any discounted customer (Eptura is ~20% under retail). Use the negotiated price-sheet / FOCUS `EffectiveCost`. This is the "accuracy upgrade" your own Summary already flags.

## App Service — what changes from today's output

You keep your whole engine — App Service now ships at **shared scope** (like VM/SQL). The changes:

1. **Output JSON, not Excel** — same content, the shape above. One file per service run.
2. **Roll up plans → one rec per `(billing scope, region, SKU)`** — the buyable unit isn't per plan *or* per sub; a shared reservation floats across every plan of that SKU across the billing scope's subs. Aggregate them, **size the quantity on merged usage (not the sum of per-plan floors)**, sum the costs, and list the plans in `evidence.contributing_plans[]`. Emitting per-plan collides on one id — the duplicate-id bug.
3. **Merge the two sheets by need** — Min-Wastage and Advance-Saving are **not** two recommendations. For each `(billing scope, region, SKU)` need, emit **one** item; put the min-wastage sizing and (if present) the advance sizing in its `sizings[]`.
4. **Region → canonical name** — `Central US` → `centralus`, `UK West` → `ukwest`, `Southeast Asia` → `southeastasia`. (Same idea as AWS `us-east-1`.) It's part of identity, so it must match exactly every run.
5. **Monthly figures** — costs are monthly. Per sizing, make them reconcile: `effective_od_monthly_cost − projected_commit_monthly_cost = estimated_monthly_saving`.
6. **Customer-effective cost** (rule #4) — swap the retail estimate for the negotiated rate.
7. **Scope + currency** — `scope_type: shared` + `scope_id` = the billing profile ID (from FOCUS/cost data, not the sub GUID); header `subscriptions_covered` = every sub you scanned; **`currency` on each recommendation** (the billing scope's currency) — a run can span billing profiles in different currencies, so it's per-rec, not a header field.

> **Not in v1 — management-group pooling**
> At shared scope, usage already pools **within one billing scope** (Eptura has 16 profiles). Pooling *across* billing scopes needs a management-group-scope reservation, which Raghu Sharma is handling later. **Not for this version** — keep one rec per billing scope.

## VM & SQL — same file, same rules, these differences

VM and SQL already emit **shared-scope, cross-sub aggregated** output, so they need *less* change than App Service. Same header, same `sizings[]`, same 4 rules (above). Two things differ from App Service for **both** of them:

- **Scope = `shared`.** `scope_type: "shared"`, and `scope_id` = the **billing profile ID** (MCA — Eptura) or **billing account / enrollment ID** (EA — Itron). Get it from the same FOCUS/cost data you already read for costs — it carries `BillingProfileId` (native) / `x_BillingProfileId` (FOCUS) per row, keyed to subscription. No billing access needed. One rec per `(billing scope, region, SKU family)`. (Bare ID only — no ARM path.)
- **Family = the ISF flexibility group** from the Reservations Catalogs API (e.g. `Dsv5 Series`, `General Purpose Gen5`), not the SKU itself. Method + pull scripts: [Azure reservation SKU families (ISF) reference](isf-sku-families-reference.md).

### VM (`all_accounts_ri.xlsx`)

| Your column | JSON field | Note |
|---|---|---|
| `Flexibility Group` (`BS Series High Memory`) | `normalized_sku_family` | already the ISF group ✓ |
| `Region` (`uksouth`) | `region` | already canonical ✓ |
| `Σ Ratio Units` | `sizings[].quantity` | normalized ratio units ✓ |
| `Term` (`1yr`) | `recommended_term` = `P1Y` | format |
| negotiated PAYG (= `Reservation Cost/yr` + `Net Saving/yr`) | `effective_od_monthly_cost` | ÷ 12 → monthly, top-level |
| `Reservation Cost / yr` | `sizings[].projected_commit_monthly_cost` | ÷ 12 → monthly |
| `Net Saving / yr (Negotiated)` | `sizings[].estimated_monthly_saving` | ÷ 12 → monthly |
| `Billing currency` | `currency` | **per rec** (billing scope's currency) |
| `Windows?`, `VMs`, `Utilisation %` | `evidence{}` | optional |

- **Add `tenant_id`** — the VM summary doesn't carry it today (App Service + SQL do).
- **One sizing** — VM emits a single quantity → `sizings` has one `min_wastage` entry.
- **`RI Price Source`** is `retail (fallback)` today → switch to the **negotiated** rate (rule #4). PAYG is already negotiated.

### SQL (`sql_reservation_*.xlsx`)

| Your column | JSON field | Note |
|---|---|---|
| `Tier` + `Family` + `Zone Redundant` | `normalized_sku_family` (e.g. `SQL DB · General Purpose Gen5 ZR`) | ⚠️ **fold all three in** — see below |
| `Region` | `region` | canonical ✓ |
| `Recommend vCores` | `sizings[].quantity` | vCores (1 vCore = 1 unit) |
| `Floor` / `Break-even` | two `sizings`: `min_wastage` / `advance` | already both on one row ✓ |
| `PAYG $/vCore/hr` × vCores × 730 | `effective_od_monthly_cost` | monthly on-demand, top-level |
| `Reserved $/vCore/hr` → monthly | `sizings[].projected_commit_monthly_cost` | |
| `Monthly Saving (USD)` | `sizings[].estimated_monthly_saving` | already monthly ✓ |
| (billing scope currency) | `currency` | **per rec** — convert from USD, see below |
| `Term` (`1y`) | `recommended_term` = `P1Y` | format |

- ⚠️ **`normalized_sku_family` must include deployment type + zone-redundancy** — `vCore` and `vCore ZR` are *separate* reservation types, and SQL DB vs Managed Instance differ. Don't leave `Zone Redundant` as a loose column, or a ZR and non-ZR rec collapse onto one id.
- ⚠️ **Currency** — SQL emits **USD** today; it must emit the billing scope's currency (**GBP** for Eptura) on **each recommendation**, or costs can't be summed across services.
- **DTU-based** databases can't be reserved (vCore model only) — skip them.

## Log Analytics / Sentinel tiers — proposed (not yet v1)

For the LA-tier script you're building. Same file shape, same 4 rules — but the commitment
here is a **workspace pricing tier** (a SKU setting with a 31-day downgrade right), not a
reservation, so identity and a few enums differ. Design is proposed, pending pipeline-side
agreement — spec in the contract's Open list.

**Identity — the tier binds to one workspace, it doesn't pool:**

| Field | Value | Note |
|---|---|---|
| `scope_type` | `resource` | new enum value |
| `scope_id` | the workspace's **full ARM resource ID**, lowercased | deliberate exception to the bare-ID rule — a workspace path isn't reconstructable, and the "purchase" is an ARM PATCH on it |
| `service` | `log_analytics` \| `sentinel` | one run (file) per service, like the others |
| `normalized_sku_family` | `commitment-tier` (constant) | scope_id already makes the row unique |
| `region` | workspace region, canonical | |

**Action fields:** `recommended_commitment_type: "commitment_tier"`, `recommended_term: null`
(no term). `sizings[].quantity` = the recommended **tier level in GB/day** (e.g. `1000`) —
re-tiering updates the same row. `quantity: 0` = "no tier / PAYG is right". One `min_wastage`
sizing; no `advance`.

**Sizing rules — each of these bit us live on Itron:**

1. **Pricing regime is per-workspace, from billing meters.** Classic = separate LA ingestion
   meter + Sentinel "…Analysis" meters → emit **two rows** (an `log_analytics` and a
   `sentinel` one). Simplified = commitment-tier meters under meter category `Sentinel`, no
   LA ingestion meter → emit **only the `sentinel` row**. One tenant can mix both regimes.
2. **Size on Analytics-tier volume only** — exclude Basic/Auxiliary Logs before computing the
   series (Itron's big workspace: 4.1 TB/day total, only ~1.4 TB/day tier-eligible).
3. **Billing meters are the source of truth for volume and rates.** The ARM table-plan API
   contradicted the meters at Itron (`plan: null` on tables billing as Basic), and raw
   `Usage` totals include non-eligible volume. Daily meter quantities literally encode the
   volume: `qty × tier-unit-GB = GB billed that day` at the tier's effective rate.
4. **FOCUS gotcha:** Sentinel never appears as `ServiceName = 'Sentinel'` — it bills under
   `'Azure Monitor'`; filter on `x_SkuMeterCategory = 'Sentinel'`.
5. **Rates are customer-effective** (rule #4 as usual) — derive $/GB from the meters
   themselves (Itron's EA is 22% below retail), not the Retail Prices API.
6. **One workspace can carry multiple tier meters for different streams** (Itron: Syslog on
   its own 50 GB/day tier beside the main 1000). Treat them as separate streams — don't sum
   them into one commitment, and flag them for review rather than auto-recommending.

## Where to drop the file

Write each run file to the **same S3 bucket as the Tuner handoff**, under a separate **`commit/` folder** (Tuner keeps its own prefix — same bucket, different folder). Use the same write credentials/IAM as the Tuner plumbing. The pipeline picks it up on a **weekly schedule** — no event/notification to emit (unlike Tuner, Commit doesn't need the RabbitMQ signal).

## Related

- [Azure Commit recommendation output contract](output-contract.md) — the full contract.
- [Azure reservation SKU families (ISF) reference](isf-sku-families-reference.md) — for **compute/SQL** families (App Service has none).
- Azure work tracker — where this sits.
