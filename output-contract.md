> Published from CloudKeeper's Azure Commit working docs on 2026-07-06. Auto-generated copy — don't edit this file; changes are overwritten on the next publish. Questions / change requests → Raghu Sharma.

# Azure Commit recommendation output contract

The format Raghu Sharma's team produces Azure Commit recommendations in, so the pipeline can consume them. Recommendation *generation* is delegated to the team; the *pipeline* (DB ingest, savings-delivered) is ours. This contract is the line between them — the team produces to it, the pipeline reads from it, and the two build independently.

Same idea as Rohit Pant's Tuner HLD, which defines the format the team produces *Tuner* output in. The difference: for Tuner, Rohit wrote the contract; for Commit, there is no Rohit, so we write it.

> **Tuner is not in this doc**
> The team produces Tuner output directly in Rohit's HLD §5.1 format → S3 → RabbitMQ. That contract already exists (`~/Downloads/Azure-Recommendation-HLD.md`); we don't re-wrap it. This doc is **Commit only**.

## Two parts: a header and a list

Each run the team produces is one file: a **header** (what this run is) and a **list of recommendations**.

One run covers **one service across a set of subscriptions** — `subscriptions_covered` lists exactly which ones.

**v1 — every service ships at `shared` scope** (`scope_type: "shared"`). Scope is per-recommendation, keyed to the **billing scope** the shared reservation is bounded by: the **billing profile ID** (MCA, e.g. Eptura) or **billing account / enrollment ID** (EA, e.g. Itron). One rec per `(billing scope, region, SKU)`.

- **VMs and SQL** — already aggregated across subs by their engines.
- **App Service** — also `shared` (confirmed 2026-07-03). A reservation floats across every plan of that SKU in the scope, so shared pools it across subscriptions instead of stranding it per-sub — higher utilisation, fewer reservations. The engine rolls all plans of the same SKU up to one rec (see App Service note below).

> **Where the shared `scope_id` comes from — no billing access needed**
> A shared reservation only pools **within one billing scope**, so shared recs are grouped per billing scope, not lumped into one "shared". The billing scope ID rides along in the **FOCUS / Cost Management data already pulled for the cost basis** (rule #4): the MCA cost export carries `BillingProfileId` (and `BillingAccountId`) per usage row keyed to subscription; FOCUS exposes it as `x_BillingProfileId`. Build the **subscription → billing scope** map from that data and tag each rec's `scope_id`. Usage/sizing that comes from another source (Resource Graph, metrics) joins the map in — a sub belongs to exactly one billing scope. No `Microsoft.Billing` role required.
>
> **Store the bare ID**, not the full ARM/billing path — `scope_type` says how to read it, and the boilerplate prefix is constant per customer (one billing account per tenant), so the full path is reconstructable if a purchase step ever needs it. Keeps `scope_id` uniform across scope types (subscription is already the bare GUID).

`subscriptions_covered` lists every sub the run scanned (completeness boundary + audit); it can be all subs or a subset, and withdrawal is bounded to it so a partial/team-sliced run stays self-contained.

> **v2 — pool at management-group scope**
> Buying at **MG scope** lets staggered usage pool across *billing scopes* into fewer reservations (shared scope only pools within one billing profile — Eptura has 16 — and can cross tenants; **MG** scope is tenant-bound). Deferred — v1's min-wastage floor already removes most waste, and all three services already pool within their billing scope at shared scope. The `management_group` enum value below is reserved for this.

```jsonc
{
  // ---- header: what this run is ----
  "version": "1.0",
  "run_id": "2026-06-25T02-14-07Z_eptura_compute",   // unique per run — full UTC timestamp to seconds (multiple runs/day must not collide); filename-safe, no colons
  "tenant_id": "adb48caa-...",                     // the customer
  "service": "compute",                            // 'compute' | 'sql' | 'appservice'
  "subscriptions_covered": [                       // every subscription this run EXAMINED — completeness is judged against this set
    "1111...", "2222...", "3333..."
  ],
  "generated_at": "2026-06-25T02:14:07Z",

  "status": "success",            // 'success' | 'failure'
  "is_complete": true,            // true = the FULL live set for this service, across subscriptions_covered; see below
  "count": 12,                    // must equal recommendations.length on success

  // ---- the recommendations ----
  "recommendations": [ /* one item per need — below */ ]
}
```

### One recommendation

One item per *need*: a normalized SKU family, in a region, at a scope. **Not per-resource** — with Instance Size Flexibility you reserve a quantity of a family, not coverage for one named VM. This is the `recommendation` target from [Azure Commit recommendation system-of-record design](system-of-record-design.md).

```jsonc
{
  // ---- identity: who/where this reservation is for (5 fields) ----
  "tenant_id": "adb48caa-...",            // the customer
  "scope_type": "shared",                 // v1: all services 'shared'. enum: 'shared' | 'management_group' | 'subscription' | 'resource_group'
  "scope_id": "XXXX-XXXX-XXX-XXX",        // just the ID, no ARM/billing path. shared → billing profile id (MCA) / billing account id (EA); subscription → sub GUID; MG → MG name; RG → RG name
  "region": "westeurope",                 // Azure's canonical region NAME (not "West Europe")
  "service": "compute",                   // 'compute' | 'sql' | 'appservice'
  "normalized_sku_family": "Dsv5",        // compute/SQL: the ISF family. App Service: the SKU itself (no ISF) — e.g. "P3V3 Windows"

  "title": "Reserve Dsv5 compute (West Europe)",
  "recommended_commitment_type": "reserved_instance",  // 'reserved_instance' | 'savings_plan'
  "recommended_term": "P1Y",              // 'P1Y' | 'P3Y'
  "currency": "GBP",                      // the billing scope's currency — governs every cost on this rec. per-rec, NOT header (billing profiles can differ)
  "effective_od_monthly_cost": 4200.00,   // customer's real rate (FOCUS), NOT retail — same baseline for all sizings

  // ---- one or two sizings for THIS need: the conservative floor, and the aggressive bar ----
  "sizings": [
    { "strategy": "min_wastage",          // ALWAYS present. the floor — reserve only what's used above the uptime bar.
      "quantity": 8.0,                    // normalized units (compute/SQL) or instance count (App Service). 0 = "don't buy at the floor" (send 0, don't omit the rec)
      "projected_commit_monthly_cost": 2940.00,
      "estimated_monthly_saving": 1260.00 },
    { "strategy": "advance",              // OPTIONAL — the break-even bar; omit if the engine didn't compute one
      "quantity": 12.0,
      "projected_commit_monthly_cost": 4100.00,
      "estimated_monthly_saving": 1500.00 }
  ],

  // ---- optional: evidence behind the call (nice to have, not required) ----
  "evidence": {
    "contributing_resource_ids": ["...", "..."],
    "coverage_window_days": 30,
    "preexisting_coverage_subtracted": 2.0   // sized against usage not already covered
  }
}
```

The team does **not** send an id. The pipeline builds it from the five identity fields (`tenant + scope + region + service + sku_family`). The team's only job is to **spell those five fields the same way every run** — same region name, same SKU-family name — so the same need lands on the same row. `commitment_type` and `term` are the recommended *action*, not identity; they can change between runs without it being a different need.

**One row per need, not per strategy.** The engine produces up to two sizings for the same need — a conservative `min_wastage` floor and an aggressive `advance` bar. They go in the `sizings[]` array **on the same recommendation**, not as separate rows. (SQL already emits both on one row — `Floor` and `Break-even` columns; App Service splits them across two sheets, but they're the same need.) `min_wastage` is the primary; `advance` is optional upside. The frontend picks which to show; the pipeline never double-counts a need.

**`min_wastage` is mandatory — send `0`, never omit (confirmed 2026-07-03).** Every recommendation carries a `min_wastage` sizing. When the conservative floor is "buy nothing," send `quantity: 0` — do **not** drop the recommendation or the sizing. `recommended_quantity` is taken from `min_wastage`, so a floor of 0 stores as `recommended_quantity = 0`: the row is kept and tracked, but **the actionable list the customer sees is `recommended_quantity > 0`** — zeros are hidden (they carry only aggressive `advance` upside, display-only). Sending 0 rather than omitting keeps the row stable across runs and avoids a false "withdrawn". `sizings` may still have one entry (just `min_wastage`) when the engine computed no `advance` bar.

## Four things the team must get right

These are what the pipeline (and the **invoice**) depend on. They're in the contract so the output is right at the source.

1. **`is_complete: true` means the run is the *whole* live set** for this service across `subscriptions_covered` — not a delta, not a page. The pipeline treats a recommendation that was there last run and is gone now as **resolved/withdrawn** — but **only for subscriptions in this run's `subscriptions_covered`**. A subscription you didn't scan this run is left untouched, never withdrawn. So if a few subscriptions fail to scan, just leave them out of `subscriptions_covered` (still `is_complete: true` for the rest) — you don't need to fail the whole run. If you can't produce the full set even for the covered subs, send `is_complete: false` and the pipeline skips withdrawal entirely.
2. **`status: "failure"` ⇒ empty list + a reason.** A failed run is *not* "everything disappeared." The pipeline skips it entirely. (Without this, a crashed generator reads as "all savings delivered" → wrong invoice.)
3. **One run is all-or-nothing.** No half-written files.
4. **Costs are the customer's real rate**, from their FOCUS export (`EffectiveCost`/`ContractedCost`), **not** the Retail Prices API. Billing 25% off retail would overcharge any discounted customer (e.g. Eptura ~20% sub-retail). See [cost basis](system-of-record-design.md). **Currency is per-recommendation, not per-run** — it's the billing scope's currency (from the same FOCUS data as `scope_id`). One run can span billing profiles in different currencies, so each rec carries its own `currency`; the pipeline must not assume a single run-level currency.

The fifth thing is on **us**, not the team: build the id consistently in the pipeline from those five fields, and pin the SKU-family naming so the team spells it the same way.

> **ISF applies to VMs and SQL — not App Service**
> `normalized_sku_family` for **compute (VMs)** and **SQL** is the Instance Size Flexibility group from the Reservations Catalogs API (e.g. `Dsv5 Series`, `General Purpose Gen5`) — see [Azure reservation SKU families (ISF) reference](isf-sku-families-reference.md). **App Service has no ISF**: reservations are SKU-specific, so there `normalized_sku_family` is just the SKU itself (e.g. `P3V3 Windows`) and `quantity` is a plain instance count. Don't run the ISF pull for App Service.
>
> **One rec per `(billing scope, region, SKU)` — roll up plans.** An App Service reservation isn't tied to a plan; it floats across every plan of that SKU in the (shared) scope. So aggregate all App Service plans of the same SKU within the billing scope into **one** recommendation, sized on the **merged** usage across those plans — *not* the sum of the per-plan floors (peaks don't align, so summing over-reserves). List the plans in `evidence.contributing_plans[]`. Don't emit per-plan rows — they share one id and would overwrite each other (the loader now rejects a file that contains such duplicates).
>
> **SQL identity carries more.** A SQL reservation is pinned by deployment type **and** zone-redundancy on top of tier+hardware. Both must be inside `normalized_sku_family` — e.g. `SQL DB · General Purpose Gen5` vs `SQL DB · General Purpose Gen5 ZR` are *different reservation types* and must not collapse to one row.

> **Existing-commitment scans must also read workspace pricing tiers (LA / Sentinel)**
> Log Analytics and Sentinel commitments are **workspace SKU settings** — `properties.sku.name = "CapacityReservation"` + `capacityReservationLevel` on the workspace resource — not `Microsoft.Capacity` reservation orders or `Microsoft.BillingBenefits` savings plans. A pre-existing-coverage scan (`evidence.preexisting_coverage_subtracted`) that reads only the Reservations/Savings-Plan APIs misses them entirely, and their charges also don't surface in FOCUS `CommitmentDiscount*` columns (they bill as plain "Commitment Tier" usage meters). Detect via Resource Graph on workspace SKUs. Caught live on Itron 2026-07-05: 4 tiered workspaces (incl. a 1000 GB/day tier), zero visible in the reservation inventory.

## Versioning

`version` bumps when the format changes. The pipeline rejects a `version` it doesn't know and quarantines the file rather than half-reading it. Adding optional fields doesn't need a bump.

## Open

- [x] **`normalized_sku_family` list** — resolved: adopt Azure's `InstanceSizeFlexibilityGroup` names from the Reservations Catalogs API (not the deprecating CSV). Compute + SQL both covered. Full method, examples, and contract mapping in [Azure reservation SKU families (ISF) reference](isf-sku-families-reference.md). *(team-facing)*
- [x] **Completeness scope** — resolved: **per service, across `subscriptions_covered`**. One run is tenant-wide for one service and spans many subscriptions; it's the full live set for every sub it lists. Withdrawal applies only to listed subs, so a failed scanner (one service, or one subscription) never withdraws anything it didn't scan. *(confirmed 2026-06-29; multi-sub header added 2026-06-30)*
- [x] **Transport** — resolved: **S3, scheduled weekly pickup**. The team writes the run file to the **same S3 bucket as Tuner's handoff, under a separate `commit/` folder** (Tuner keeps its own prefix). The pipeline reads it on a weekly schedule — **no RabbitMQ event** (unlike Tuner; Commit data changes slowly, so polling is enough). Exact bucket + write IAM are the same as the Tuner plumbing. *(confirmed 2026-06-30)*
- [x] **Identity hash** — resolved (2026-07-01): `sha256` of the five identity fields in fixed order `tenant_id | scope_id | region | service | normalized_sku_family`, pipe-separated, each `trim`+`lower`; hex digest is the PK. Lowercasing absorbs cosmetic casing drift; *semantic* drift (`Dsv5` vs `Dsv5 Series`) still splits rows, so the team's spell-it-the-same responsibility stands. Internal to the pipeline. *(design: [Azure Commit recommendation system-of-record design](system-of-record-design.md) → Decisions locked 2026-07-01)*
- [x] **Shared `scope_id`** — resolved (2026-07-01): the **billing scope** — billing profile ID (MCA) / billing account (EA), sourced from the FOCUS / Cost Management data already pulled for the cost basis (`BillingProfileId` / `x_BillingProfileId`, keyed to subscription). No billing role needed. Shared recs grouped per `(billing scope, region, SKU)`.
- [ ] **Verify: shared `scope_id` source** — confirm the FOCUS / Cost Management data we pull actually populates `BillingProfileId` (MCA) / `BillingAccountId` (EA) per usage row. Certain if the customer's FOCUS export is billing-account-scoped; worth a check if cost is queried per-subscription. *(v1 dependency for shared-scope recs)*
- [x] **App Service scope** — resolved (2026-07-03): **`shared`, keyed by billing scope**, same as VM/SQL — *not* per-subscription. A reservation floats across every plan of that SKU in the scope, so shared pools it across subs (higher utilisation). The engine rolls all plans of a SKU up to one rec, sized on merged usage. *(caught on the first live run — Itron/Eptura App Service files came in as `shared`)*
- [x] **`min_wastage` mandatory** — resolved (2026-07-03): always send a `min_wastage` sizing; use `quantity: 0` for "don't buy at the floor" rather than omitting it. `recommended_quantity` = the `min_wastage` quantity; the actionable list is `> 0`; zeros are stored but hidden. *(caught on the first live run — 18/94 Eptura recs arrived with only `advance`)*
- [ ] **LA / Sentinel tiers as a Commit service** — the commitment vehicle is a workspace pricing tier (31-day downgrade right, no term, no purchase transaction), so `recommended_commitment_type` / `recommended_term` don't fit as-is; needs a contract extension before LA tier recommendations productionise. Engine already exists (`war-azure-log-analytics-commitment`). *(added 2026-07-05 after the Itron LA run)*
- [ ] **v2: MG-scope pooling** — move from per-billing-scope shared to **management-group-scope** purchasing, so usage pools across billing scopes (shared pools only within one; Eptura has 16). Sizing stays on merged usage; MG scope is tenant-bound. *(deferred — v1's min-wastage floor already removes most waste, and all three services already pool within their billing scope)*

## Related

- [Azure Commit recommendation system-of-record design](system-of-record-design.md) — the model this contract serializes (identity, target/holding split, cost basis).
- Azure Commit summary table schema — the terminal sink the pipeline writes to.
- Azure work tracker — where this sits in the workstream.
- Tuner equivalent: Rohit Pant's HLD (`~/Downloads/Azure-Recommendation-HLD.md`) — the team produces to it directly; not covered here.
