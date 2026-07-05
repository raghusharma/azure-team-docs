> Published from CloudKeeper's Azure Commit working docs on 2026-07-05. Auto-generated copy — don't edit this file; changes are overwritten on the next publish. Questions / change requests → Raghu Sharma.

# Azure reservation SKU families (ISF) reference

The canonical source for the `normalized_sku_family` field in the [Commit contract](output-contract.md). Reservations apply to a **flexibility group**, not one SKU — Instance Size Flexibility (ISF). The team adopts Azure's own group names; we don't invent them.

## Bottom line for the team

- `normalized_sku_family` = Azure's **`InstanceSizeFlexibilityGroup`** name, used verbatim (e.g. `Dsv5 Series`, `Esv5 Series`, `FSv2 Series`). Casing varies by group — copy it exactly as the API returns it.
- `recommended_quantity` = total footprint in **ratio units**, normalized so the smallest SKU in the group = 1. Sum `count × ratio` across running sizes in the group.
- **Pull both compute and SQL from the same place — the Reservations Catalogs API.** Don't hardcode the old CSV (see deprecation below).

## Source of truth: Catalogs API (not the CSV)

> **The ISF ratio CSV is being retired**
> The CSV at `aka.ms/isf` **stopped updating 2026-05-09** and is **removed 2026-08-30**. Build against the API instead — the data is the same, the CSV is just going away.

ISF lives in each catalog item's `skuProperties[]` array as name/value pairs:
- `ReservationsAutofitGroup` → the flexibility group name
- `ReservationsAutofitRatio` → the ratio

(MS docs are inconsistent — some responses name these `InstanceSizeFlexibilityGroup` / `InstanceSizeFlexibilityRatio`. Read **either**.)

**Bash (`az` CLI + `jq`)** — the CLI paginates for you, so no `nextLink` handling:
```bash
az extension add --name reservations 2>/dev/null   # one-time
SUB="<subscription-id>"; LOC="eastus"
RTYPE="VirtualMachines"   # | SqlDatabases | SqlManagedInstance | RedisCache | ...

az reservations catalog show \
  --subscription "$SUB" --reserved-resource-type "$RTYPE" --location "$LOC" \
| jq -r '
  ["InstanceSizeFlexibilityGroup","ArmSkuName","Ratio"],
  ( .[]
    | (reduce (.skuProperties // [])[] as $p ({}; .[$p.name] = $p.value)) as $props
    | ($props.ReservationsAutofitGroup   // $props.InstanceSizeFlexibilityGroup) as $g
    | ($props.ReservationsAutofitRatio   // $props.InstanceSizeFlexibilityRatio) as $r
    | select($g and $r)
    | [ $g, (.skuName // .name), $r ] )
  | @csv'
```

**Python (`azure-mgmt-reservations` + `azure-identity`)** — includes the smallest-SKU-=1 normalization:
```python
from collections import defaultdict
from azure.identity import DefaultAzureCredential
from azure.mgmt.reservations import AzureReservationAPI

client = AzureReservationAPI(credential=DefaultAzureCredential())

def isf_ratios(subscription_id, reserved_resource_type, location):
    catalog = client.reservation.get_catalog(
        subscription_id=subscription_id,
        reserved_resource_type=reserved_resource_type,   # "VirtualMachines", "SqlDatabases", ...
        location=location,
    )
    groups = defaultdict(list)
    for item in catalog:
        props = {p.name: p.value for p in (item.sku_properties or [])}
        group = props.get("ReservationsAutofitGroup") or props.get("InstanceSizeFlexibilityGroup")
        ratio = props.get("ReservationsAutofitRatio") or props.get("InstanceSizeFlexibilityRatio")
        if group and ratio:
            groups[group].append((item.name, float(ratio)))

    rows = []   # (group, arm_sku_name, normalized_ratio)  — smallest SKU in each group = 1
    for group, skus in groups.items():
        lo = min(r for _, r in skus)
        rows.extend((group, sku, round(r / lo, 4)) for sku, r in skus)
    return rows
```

Pure-`requests` alternative (if you'd rather not add the SDK): GET
`https://management.azure.com/subscriptions/{sub}/providers/Microsoft.Capacity/catalogs?api-version=2022-11-01&reservedResourceType={type}&location={loc}`
with a bearer token, then walk `value[].skuProperties[]` the same way — but **handle `nextLink` yourself**.

Notes the team needs:
- **Permission:** `Microsoft.Capacity/catalogs/read`.
- **Region-specific:** ratios can vary by region → pull per target region.
- **Paginated:** follow `nextLink` for the full set.
- **Normalize:** raw ratios don't always start at 1 (e.g. `BS Series` starts 0.25, `Ddsv5 Series` starts 2). Divide each by the group's minimum so the smallest = 1, then quantities are intuitive and consistent.

## Compute (VM) — three columns

```
InstanceSizeFlexibilityGroup,ArmSkuName,Ratio
Dsv5 Series,Standard_D2s_v5,1
Dsv5 Series,Standard_D4s_v5,2
Dsv5 Series,Standard_D8s_v5,4
Dsv5 Series,Standard_D16s_v5,8
Dsv5 Series,Standard_D32s_v5,16
Dv5 Series,Standard_D2_v5,1
Esv5 Series,Standard_E2s_v5,1
Esv5 Series,Standard_E8s_v5,4
FSv2 Series,Standard_F2s_v2,1
FSv2 Series,Standard_F8s_v2,4
DSv2 Series,Standard_DS1_v2,1
DSv2 Series,Standard_DS2_v2,2
DSv2 Series,Standard_DS4_v2,8
```

A `Dsv5 Series` reservation of quantity 8 covers, e.g., 8× D2s_v5, or 2× D8s_v5, or one D16s_v5 + one D8s_v5 — anything summing to ratio 8. Different group ≠ covered (a `Dsv5 Series` reservation does not cover `Esv5 Series` usage), which is why the group is part of identity.

## SQL — same API, family encodes more

SQL has **no ratio CSV** — but it comes through the **same Catalogs API** with `InstanceSizeFlexibilityGroup` names like `General Purpose Gen5` (ArmSkuName e.g. `GP_Gen5_2`). The unit is the **vCore** (1 vCore = 1 unit; quantity = total vCores).

A SQL reservation is pinned by **Region + Deployment Type + Performance Tier + Hardware generation**, so for SQL the family must capture all of:
- **Deployment type:** SQL Database (single / elastic pool) vs SQL Managed Instance
- **Performance tier:** General Purpose / Business Critical / Hyperscale
- **Hardware:** standard-series (Gen5) / premium-series / premium-series memory-optimized

The `InstanceSizeFlexibilityGroup` from the catalog already bundles tier + hardware (e.g. `General Purpose Gen5`); the team pairs it with **deployment type** and **zone-redundancy** to form the full family. The reservation floats across vCores within that combination (Azure's "vCore size flexibility").

**Build the SQL `normalized_sku_family` as:** `<deployment type> · <catalog group>[ ZR]` — e.g. `SQL DB · General Purpose Gen5`, or `SQL DB · General Purpose Gen5 ZR` when zone-redundant. The current engine output (`sql_reservation_*.xlsx`) carries `Tier` + `Family` + a separate `Zone Redundant` column — fold all three into this one field, because:

> **Zone-redundancy and deployment type are part of identity**
> **Zone redundancy is a *separate* reservation type** — `vCore` and `vCore ZR` are bought separately and don't cover each other. So a ZR and a non-ZR recommendation for the same tier are **different needs** and must land on different rows; if `Zone Redundant` stays a loose column they'd collapse into one. Same for **deployment type** (SQL DB single/elastic pool vs Managed Instance). **DTU-based** databases (basic/standard/premium) **cannot** be reserved at all — vCore model only; flag and skip.

## How it maps to the contract

| Contract field | Compute | SQL |
|---|---|---|
| `service` | `compute` | `sql` |
| `normalized_sku_family` | `InstanceSizeFlexibilityGroup` (e.g. `Dsv5 Series`) | deployment type + group + ZR flag (e.g. `SQL DB · General Purpose Gen5 ZR`) |
| `recommended_quantity` | Σ `count × ratio`, smallest SKU = 1 | total vCores |
| `region` | catalog `Location` | catalog `Location` |

## Related

- [Azure Commit recommendation output contract](output-contract.md) — consumes `normalized_sku_family`; this resolves that open item.
- Azure Retail Prices API gotchas — separate API; note the SQL unit-normalization bug there.
- [Azure Commit recommendation system-of-record design](system-of-record-design.md) — why the family is part of the recommendation identity.
