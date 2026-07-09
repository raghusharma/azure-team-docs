> Published from CloudKeeper's Azure Commit working docs on 2026-07-09. Auto-generated copy — don't edit this file; changes are overwritten on the next publish. Questions / change requests → Raghu Sharma.

# Azure Commit holdings scan contract

The format a **holdings scan** is produced in, so the Azure Commit **commitment loader** (pipeline #2) can consume it. This is the holdings-track twin of the [output contract](output-contract.md): that one is the line between the recommendation *generator* and the ingest loader; this one is the line between the holdings *scanner* (the producer) and the commitment loader.

The loader is deliberately **decoupled** from Azure — it reads this file and MERGEs it, customer-agnostic. All the messy per-customer acquisition lives in the producer, for one reason: getting holdings out of Azure varies a lot per customer and per holding type (three different source APIs, three different RBAC/scope shapes — see [the design decision](system-of-record-design.md)), while *reconciling holdings into the ledger* is uniform. Split at that seam and the loader stays simple and testable; the variety is quarantined in the producer.

> **What the producer must acquire — three sources, one row shape**
> A holding is a **reservation**, a **savings plan**, or a **Log Analytics / Sentinel commitment tier**, from three different Azure surfaces; the producer normalizes all three into the single holding shape below. **Sourcing rule** (from the [design](system-of-record-design.md)): **FOCUS-first, live APIs optional** — a missing per-customer role *degrades precision, it never breaks the pipeline.* So build these as separate ingestions per source, each self-contained:
> - **Reservations** — primary source the `Microsoft.Capacity/reservationOrders` API (tenant-scoped **Reservations Reader**), which gives exact term/dates; **fall back to FOCUS** (`CommitmentDiscount*`) where that role isn't granted. The order carries `displayName`/`term`/`quantity`/dates/`provisioningState` **but not** region, SKU family, or applied/billing scope — the producer enriches those (per-reservation GET + the FOCUS billing-scope map).
> - **Savings plans** — **FOCUS by design** (`CommitmentDiscountName` + purchase date + `x_SkuTerm`). The `Microsoft.BillingBenefits/savingsPlanOrders` API needs a **billing-account-scoped** role we deliberately don't require — it's a separate infosec ask per customer, and today returns 403 for the `CkAzureTuner` SP. Treat the API as *optional* future enrichment for exactness, never a dependency.
> - **LA / Sentinel tiers** — the one source FOCUS can't see: a workspace SKU setting (`properties.sku.name = "CapacityReservation"` + `capacityReservationLevel`), read via **Azure Resource Graph**. Invisible to the reservations API and to FOCUS `CommitmentDiscount*` columns (they bill as plain "Commitment Tier" meters). Caught live on Itron 2026-07-05 — 4 tiered workspaces, zero in the reservation inventory.

## Two parts: a header and a list

Each scan is one file: a **header** (whose holdings, when scanned) and a **list of holdings**. One file is one **tenant**.

```jsonc
{
  // ---- header ----
  "version": "1.0",
  "scan_id": "2026-07-08T09-00-00Z_itron_holdings",  // unique per scan; filename-safe
  "tenant_id": "adb48caa-...",                        // one customer per file
  "generated_at": "2026-07-08T09:00:00Z",
  "source": "customer-pre-existing",                  // DEFAULT source for holdings that don't set their own
  "count": 4,                                          // optional; if present, must equal holdings.length

  "holdings": [ /* one item per holding — below */ ]
}
```

Unlike the recommendation contract there is **no `is_complete` / withdrawal** — holdings are never withdrawn (a reservation you bought stays bought). Idempotency is the **natural key**: the loader MERGEs on `commitment_id`, so re-running the same scan just re-upserts. A short/partial scan therefore only *under-covers*; it never corrupts. `count` is an optional truncation guard.

## One holding

```jsonc
{
  // ---- identity: the 5 fields, ALREADY normalized (loader recomputes recommendation_id from them) ----
  "tenant_id": "adb48caa-...",
  "scope_id": "XXXX",                 // the SAME scope_id the rec side uses: shared → billing scope id; resource (LA) → workspace ARM path
  "region": "eastus",                 // Azure canonical region name
  "service": "compute",               // 'compute' | 'sql' | 'appservice' | 'log_analytics' | 'sentinel'
  "normalized_sku_family": "Dsv5",    // ISF family (compute/SQL); the SKU (App Service); the GB/day tier e.g. "1000" (LA)

  // ---- classification ----
  "source": "customer-pre-existing",  // 'ck-purchased' | 'customer-pre-existing'  (omit to inherit the header default)
  "commitment_type": "reserved_instance",  // 'reserved_instance' | 'savings_plan' | 'log_analytics_commitment_tier'
  "status": "active",                 // 'active' | 'expired' | 'exchanged' | 'cancelled' — LEDGER vocabulary; producer maps Azure provisioningState to it

  // ---- Azure ids (nullable per type — an SP has no order id; an LA tier has neither) ----
  "reservation_order_id": "7e6c...",  // null for savings plans AND LA tiers
  "reservation_id": "/providers/Microsoft.Capacity/reservationOrders/7e6c.../reservations/705e...",  // savingsPlanId for SPs; null for LA
  "lease_id": "...",                  // JOIN key to the summary table. RI/SP: the reservation/plan id. LA: the workspace ARM path
  "commitment_id": null,              // OPTIONAL — the loader mints one from the ids if omitted; supply only to override

  // ---- what it is (quantity is the immutable core) ----
  "term": "P1Y",                      // 'P1Y' | 'P3Y'; null for LA (31-day rolling minimum, no fixed term)
  "quantity": 3,                      // REQUIRED. unit count (RI/SP); daily-GB tier level e.g. 1000 (LA)
  "billing_plan": "Monthly",          // 'Monthly' | 'Upfront'; null for LA (billed daily)

  // ---- who pays ----
  "billing_scope_id": "/subscriptions/aaaa-...",  // a subscription even for shared-scope EA reservations
  "billing_scope_type": "subscription",           // 'subscription' | 'billing_profile' | 'billing_account'
  "billing_account_id": null,                     // resolved EA enrollment / MCA billing account (nullable; may be filled from cost data)

  // ---- where the discount applies ----
  "applied_scope_type": "shared",     // 'shared' | 'subscription' | 'management_group' | 'resource_group'; for LA always 'resource'
  "applied_scope_id": null,           // null for 'shared'; sub/RG/MG id otherwise; for LA the workspace ARM path

  // ---- lifecycle ----
  "date_purchased": "2026-02-01T00:00:00Z",  // benefit start
  "term_end_date": "2027-02-01",             // YYYY-MM-DD; null for LA (no fixed term)
  "amortized_upfront_cost": null,

  "details": { "provisioningState": "Succeeded" }  // raw payload → ledger `details` VARIANT
}
```

The producer does **not** send `recommendation_id`, `billable`/`billable_reason`/the fee window, or `realized_monthly_saving` — the loader recomputes the id and derives billing from `source`; `realized_monthly_saving` is filled later by the realized-savings loader (#3).

## Four things the producer must get right

1. **Spell the five identity fields the same way the recommendation side does.** The loader recomputes `recommendation_id = sha256(tenant_id, scope_id, region, service, normalized_sku_family)` to link a holding to the rec it covers. A family or scope spelled differently on the two tracks silently fails to match → the holding stores a `recommendation_id` that hits no rec, and residual/coverage looks wrong. Same "spell-it-the-same" rule as the output contract, on the holdings side.
2. **`status` is the ledger vocabulary, mapped from Azure — and the scan is the full enumeration.** Emit `active | expired | exchanged | cancelled`, mapping Azure `provisioningState` (`Cancelled → cancelled`, `Succeeded within term → active`). Include **terminal-state holdings** in the scan (don't filter to active-only) — the loader reads status, it never infers it from a holding being absent. `exchanged` only the producer can know (Azure shows an exchange as `Cancelled` + a new SP order), so set it from CK records.
3. **`commitment_id` must be stable and unique per holding.** Omit it and the loader mints from the Azure id (reservation/plan id, or workspace path for LA). Two holdings resolving to the **same** `commitment_id` in one scan is a producer bug — the loader rejects the whole file (they'd self-overwrite via MERGE).
4. **The core is immutable — don't restate it wrongly on a re-scan.** `quantity` / `term` / `date_purchased` are frozen at first load; a later scan may refresh `status` / `billing_account_id` / `details`, but the loader ignores changes to the frozen fields. So a scan that mis-reports a bought quantity can't corrupt history — but get it right the first time.

## Versioning

`version` bumps when the format changes; the loader rejects an unknown `version` and quarantines the file rather than half-reading it. Adding optional fields doesn't need a bump.

## Open

- [ ] **Transport** — same as the output contract's: a scheduled S3 pickup. Confirm the bucket + a `holdings/` prefix (separate from `commit/`). The loader's `pull` already reads a prefix.
- [ ] **`lease_id` join field** — which Azure id the FOCUS export joins on (reservationId vs reservationOrderId) is TBD; store the chosen one as `lease_id`.
- [ ] **LA `workspaces_covered`** — the resource-scope completeness signal (mirrors `subscriptions_covered`); only needed if we ever want to detect a genuinely-gone tier, and holdings don't withdraw anyway, so low priority.

## Related

- [Azure Commit recommendation system-of-record design](system-of-record-design.md) — the model this serializes (the target/holding split, `commitment_ledger`, the 2026-07-09 loader decisions).
- [Azure Commit recommendation output contract](output-contract.md) — the advice-track twin (the recommendation generator → ingest loader).
- Azure Commit summary table schema — where `lease_id` joins to per-resource realized savings (#3).
- [Azure reservation SKU families (ISF) reference](isf-sku-families-reference.md) — the `normalized_sku_family` naming both tracks must share.
