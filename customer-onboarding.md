> Published from CloudKeeper's Azure Commit working docs on 2026-07-07. Auto-generated copy — don't edit this file; changes are overwritten on the next publish. Questions / change requests → Raghu Sharma.

# CloudKeeper Azure — Customer Onboarding Guide

This guide grants CloudKeeper the access it needs to give you visibility into your Azure spend and to deliver cost-optimization and reservation / savings-plan recommendations.

Steps 1–4 grant the baseline read access; Steps 5–6 set up daily exports of your cost data that CloudKeeper reads on an ongoing basis; purchase execution is authorized separately later (see [A8](#a8--authorising-purchases-later)).

**All access granted during onboarding is read-only.** Purchase, exchange, and refund execution requires a separate authorization step, done per deal (see [A8](#a8--authorising-purchases-later)). Your raw cost data stays in your own tenant.

**Estimated time:** 30 minutes.

> Prerequisites, CLI alternatives, security details, special cases, and FAQs are in the [Appendix](#appendix). The main body covers the portal procedure only.

---

## Step 1 — Consent to the CkAzureTuner application

Open this URL in a browser signed in as a **Global Administrator** of your tenant:

```
https://login.microsoftonline.com/<your-tenant-id>/adminconsent?client_id=17567d2e-c1ee-49c6-81ab-83500cbe741a
```

Replace `<your-tenant-id>` with your Microsoft Entra tenant ID, review the requested permissions, and click **Accept**.

## Step 2 — Reader at the root management group

**Management groups** → **Tenant Root Group** → **Access control (IAM)** → **+ Add role assignment** → role `Reader` → member `CkAzureTuner` → **Review + assign**.

## Step 3 — Cost Management Reader

**Management groups** → **Tenant Root Group** → **Access control (IAM)** → **+ Add role assignment** → role `Cost Management Reader` → member `CkAzureTuner` → **Review + assign**.

This is standard cost-data RBAC at management-group scope — it grants view-only access to Cost Analysis, usage, forecasts, and budgets, and works the same regardless of your billing arrangement.

## Step 4 — Reservations Reader (all reservations in the tenant)

A reservation is a tenant-level resource with its own access control, independent of your subscriptions and of your billing account — so this grant is made at **tenant scope**.

**Home** → **Reservations** → **Role assignment** (top navigation) → role `Reservations Reader` → member `CkAzureTuner` → **Save**.

> Granting at tenant scope requires the person making the assignment to hold **User Access Administrator** at the tenant root — granted via *elevate access* (**Microsoft Entra ID → Properties → Access management for Azure resources**).

## Step 5 — Configure the Cost Management Export

**Cost Management + Billing** → your billing account → **Exports** → **+ Create**.

On **Basics**, select the **All data** template (actual + amortized + FOCUS) → **Next**.

On **Datasets**, set **Export prefix** to `cloudkeeper`. The three datasets are preselected and already run **Daily** — leave them as-is → **Next**.

On **Destination**, choose **Create new**, then pick any **Subscription**, **Resource group**, and **Location** in your own tenant. Set the rest as:

| Field | Value |
| --- | --- |
| Account name | `cloudkeeperexport` (append digits if taken — it must be globally unique) |
| Container | `cost-exports` |
| Directory | `azure-cost` |
| Format | **Parquet** *(default is CSV — change it)* |
| Compression type | **Snappy** *(default is Gzip)* |
| Overwrite data | leave **enabled** (the default) |

Select **Review + create**. The first files land within 4 hours.

*Already export these, or is the storage account behind a vNet / private endpoint? See [A5](#a5--special-cases) before continuing to Step 6.*

## Step 6 — Storage Blob Data Reader on the export storage account

Open the storage account from Step 5 → **Access control (IAM)** → **+ Add role assignment** → role `Storage Blob Data Reader` → member `CkAzureTuner` → **Review + assign**.

## Step 7 — Notify CloudKeeper

Email your CloudKeeper contact with:

- Microsoft Entra **tenant ID**
- **Billing account ID** (Cost Management + Billing → Properties)
- Service principal **objectId** for `CkAzureTuner` (lookup in [A4](#a4--cli-verification))
- Storage account **resource ID** (or name + subscription)
- The export **prefix** / **container** / **directory** *only if you changed them* from the Step 5 defaults (`cloudkeeper` / `cost-exports` / `azure-cost`)

CloudKeeper will pull a sanity-check file and confirm onboarding is complete, typically within one business day.

---

## Verification checklist

Run through this before sending the Step 7 email. CLI scripts to verify each item are in [A4](#a4--cli-verification).

- [ ] Step 1 — `CkAzureTuner` visible in **Microsoft Entra ID → Enterprise applications**
- [ ] Step 2 — `Reader` assigned to `CkAzureTuner` at root management group
- [ ] Step 3 — `Cost Management Reader` assigned to `CkAzureTuner` at root management group
- [ ] Step 4 — `Reservations Reader` assigned to `CkAzureTuner` at tenant scope (visible under **Home → Reservations → Role assignment**)
- [ ] Step 5 — export status **Active**; Parquet files visible in the three `cloudkeeper-*-cost` subfolders under `cost-exports/azure-cost/`
- [ ] Step 6 — `Storage Blob Data Reader` assigned to `CkAzureTuner` on the export storage account
- [ ] Step 7 — Notification email sent

---

## Support

For onboarding questions, contact your CloudKeeper FDE, or email support@cloudkeeper.com.

---

# Appendix

## A1 — Prerequisites

You (or someone with each of these privileges) needs to perform the corresponding steps:

- **Global Administrator** in your Microsoft Entra tenant — Step 1
- **Owner** or **User Access Administrator** on the root management group — Steps 2 and 3
- **User Access Administrator at the tenant root** (granted via *elevate access*) — Step 4, to assign tenant-scoped Reservations Reader
- **Billing account owner / administrator** on your billing account — Step 5
- **Owner** on the subscription or resource group where the export storage account will live — Steps 5 and 6 (Step 5 creates the account and lets Azure assign the export's managed identity write access; Step 6 grants CloudKeeper read access). *Contributor + User Access Administrator* together also works. If you point the export at an existing storage account instead, **Owner** or **User Access Administrator** on that account is enough.

> **Already onboarded with CloudKeeper?**
> If your tenant has previously consented to `CkAzureTuner` for another CloudKeeper service, Step 1 is already done and some role assignments may already be in place. Verify in the portal before re-assigning.

## A2 — What CkAzureTuner can and cannot do

CkAzureTuner is authorized through two independent systems:

- **Microsoft Graph API permissions** — what the Step 1 consent screen lists. The app registration requests only `User.Read` (Delegated), which lets the signed-in user read their own profile and grants the app no other access. This is the minimal default permission; the consent screen will show this single entry.
- **Azure RBAC role assignments** — Steps 2, 3, 4, and 6. Every actual capability (reading resources, cost data, reservation portfolio, export storage) is granted through explicit RBAC role assignments at specific scopes. These are granted by you, visible in the portal, and individually revocable — see [A7](#a7--revoking-access).

In effect: Step 1 provisions the service principal in your tenant with no substantive Microsoft API access. All meaningful access is granted by you in Steps 2–6 and remains under your control.

The roles granted in Steps 2–4 and Step 6 are read-only. `CkAzureTuner` can:

- List resources and read their configuration / metadata
- Read cost and usage data
- Read reservation portfolio state
- Read Parquet files from the storage account in Step 5

It **cannot**:

- Create, modify, or delete any resource
- Make any purchase or commitment on your behalf
- Read any storage account other than the one in Step 5

Purchase execution uses a different service principal (`CkAzureCommit-Purchaser`), granted just-in-time at purchase authorization — see [A8](#a8--authorising-purchases-later).

**Role-by-role summary:**

| # | Role / action | Scope | Purpose |
| - | --- | --- | --- |
| 1 | App consent: `CkAzureTuner` | Tenant | Creates the service principal we assign roles to |
| 2 | Reader | Root management group | Resource inventory and metadata |
| 3 | Cost Management Reader | Root management group | Cost & usage data — Cost Analysis, forecasts, budgets (view-only) |
| 4 | Reservations Reader | Tenant (all reservations) | Existing reservation portfolio |
| 5 | Configure a Cost Management export bundling the actual, amortized, and FOCUS 1.0r2 datasets | Billing account → your storage account | Daily cost data persisted in your tenant |
| 6 | Storage Blob Data Reader | The storage account from Step 5 | CloudKeeper's read access to your exported data |

## A3 — CLI procedures

CLI alternatives for each portal step, using `az`.

### Step 1 — Consent

No CLI equivalent. Admin consent must be granted via the portal URL in Step 1.

### Step 2 — Reader at root management group

```bash
ROOT_MG_ID=$(az account management-group list \
  --query "[?properties.displayName=='Tenant Root Group'].id" -o tsv)

SP_OBJECT_ID=$(az ad sp list --display-name CkAzureTuner --query "[0].id" -o tsv)

az role assignment create \
  --role "Reader" \
  --assignee-object-id "$SP_OBJECT_ID" \
  --assignee-principal-type ServicePrincipal \
  --scope "$ROOT_MG_ID"
```

### Step 3 — Cost Management Reader

```bash
ROOT_MG_ID=$(az account management-group list \
  --query "[?properties.displayName=='Tenant Root Group'].id" -o tsv)

az role assignment create \
  --role "Cost Management Reader" \
  --assignee-object-id "$SP_OBJECT_ID" \
  --assignee-principal-type ServicePrincipal \
  --scope "$ROOT_MG_ID"
```

### Step 4 — Reservations Reader

Best done in the portal (Step 4). Tenant-scoped reservation RBAC can also be granted with PowerShell (`New-AzRoleAssignment` for `Reservations Reader` against the reservation scope, after elevating access) — note that Microsoft documents assignments made via PowerShell as **not visible in the portal**, so verify with `Get-AzRoleAssignment`.

### Step 5 — Exports

`az costmanagement export create` creates one export per dataset type, so run it three times — `ActualCost`, `AmortizedCost`, and the FOCUS dataset (version `1.0r2`) — all pointing at the same storage account / container, each with its own directory. Refer to current Microsoft docs as the parameter shape evolves.

### Step 6 — Storage Blob Data Reader

```bash
SA_ID=$(az storage account show --name <your-export-sa> --query id -o tsv)

az role assignment create \
  --role "Storage Blob Data Reader" \
  --assignee-object-id "$SP_OBJECT_ID" \
  --assignee-principal-type ServicePrincipal \
  --scope "$SA_ID"
```

For container-scoped assignment, use `$SA_ID/blobServices/default/containers/<container-name>` as the `--scope` value.

## A4 — CLI verification

```bash
# Step 1 — service principal exists; capture the objectId for later checks
az ad sp list --display-name CkAzureTuner \
  --query "[].{appId:appId, objectId:id}" -o table

SP_OBJECT_ID=$(az ad sp list --display-name CkAzureTuner --query "[0].id" -o tsv)

# Step 2 — Reader at root management group
ROOT_MG_ID=$(az account management-group list \
  --query "[?properties.displayName=='Tenant Root Group'].id" -o tsv)
az role assignment list --assignee "$SP_OBJECT_ID" --scope "$ROOT_MG_ID" -o table

# Step 6 — Storage Blob Data Reader on the export storage account
SA_ID=$(az storage account show --name <your-export-sa> --query id -o tsv)
az role assignment list --assignee "$SP_OBJECT_ID" --scope "$SA_ID" -o table
```

Step 3 (billing-scope assignment) is verified in the portal **Access control (IAM) → Role assignments** pane of the billing account. Step 4 (tenant-scoped Reservations Reader) is verified under **Home → Reservations → Role assignment**.

## A5 — Special cases

### vNet-restricted storage accounts

If the storage account from Step 5 has **public network access disabled** or restricts traffic to a vNet, CloudKeeper cannot read it cross-tenant by default. Pause and contact your CloudKeeper FDE — we'll work out the access path (firewall allow-list, private endpoint, or API-pull fallback) before continuing to Step 6.

### Container-scoped role assignment (tighter)

For maximum isolation, assign `Storage Blob Data Reader` at container scope instead of storage-account scope. In the portal, open the container's IAM blade; in CLI, use `$SA_ID/blobServices/default/containers/<container-name>` as `--scope`.

### Export setup details (Step 5)

- **The "All data" template** bundles exactly the three cost datasets (actual, amortized, FOCUS) — not the reservation or price-sheet datasets — so the Parquet format stays available. On the Destination step the format **defaults to CSV / Gzip**; switch it to **Parquet / Snappy**.
- **Folder layout.** The template names the three exports `cloudkeeper-actual-cost`, `cloudkeeper-amortized-cost`, and `cloudkeeper-focus-cost` (from your `cloudkeeper` prefix), and each becomes a subfolder under `cost-exports/azure-cost/` — all in the single storage account, so the Step 6 grant covers all three.
- **FOCUS schema version.** The FOCUS export uses the current generally available version (**1.0r2**). To view or change it, edit that export with the pencil icon on the Datasets step.
- **File partitioning** is on by default and can't be disabled — that's expected. Each run includes a `manifest.json` describing the partitions.

### Already have a cost export

If your tenant already runs daily Cost Management exports (actual, amortized, or FOCUS 1.0r2) to a storage account in your own subscription, you can reuse them. In Step 5, create only the dataset types you don't already export, record the storage account / container / directory for each, and proceed to Step 6 — grant `Storage Blob Data Reader` on that storage account.

### Multiple billing profiles

Step 3 (root management group) and Step 4 (tenant scope) each cover the whole tenant with a single grant — you never repeat them per billing profile or per billing account. The Step 5 export is configured per **billing account**, so one billing account with many profiles needs just the one export (three datasets) covering all of its profiles. This is the normal case for a multi-profile MCA customer.

Only if your organization has **more than one billing account** does Step 5 repeat — once per billing account, all writing to the same storage account under a distinct directory per billing account. Contact your FDE before configuring multiple billing accounts.

## A6 — Frequently asked questions

**Where does our cost data live?**
Your raw cost files stay in your own storage account. CloudKeeper reads them on demand and normalizes into an analysis warehouse. You retain custody, retention control, and the ability to revoke access — see [A7](#a7--revoking-access).

**Why all three datasets — actual, amortized, and FOCUS?**
The actual and amortized datasets are Azure's native cost views (actual = cash/billed; amortized = reservation and savings-plan cost spread across the term). FOCUS is the FinOps Foundation Open Cost & Usage Specification — it carries both billed and amortized cost in one file, marks closed-month restatements explicitly (`ChargeClass = Correction`), and uses standardized commitment-discount columns. We use FOCUS **1.0r2** — the current generally available release of the FOCUS 1.0 schema (it differs from 1.0 only in using full ISO-8601 timestamps); the 1.2 schema is still in preview, so we don't use it yet.

**Why does Step 3 grant API-level cost access if Step 5 already exports the data?**
The daily export is the bulk cost feed; the Cost Management API does things the export cannot:

- **Reconciliation** — cross-check our ingested totals against your own Cost Analysis, so a dropped day or a missing meter in the export surfaces immediately.
- **Budgets** — read your configured budgets and their alert state (not carried in the export).
- **Forecasts** — Azure's spend-to-period-end projection (the exports carry actual and amortized cost, not forecast).
- **Freshness** — query today's month-to-date spend on demand, instead of waiting for the next daily export drop, so recommendations act on current data.

It also covers one-time backfill of months prior to the export's start date.

**What happens during the gap between consent and our first analysis?**
The first export file typically lands within 4 hours of Step 5. Backfill of prior months happens through a separate one-time pull that CloudKeeper triggers using the Cost Management Reader access from Step 3 — you don't need to do anything for the backfill.

## A7 — Revoking access

Remove individual capabilities by removing the corresponding role assignment:

- **Resource read** — remove Step 2's `Reader` role at the root management group
- **Cost data read** — remove Step 3's `Cost Management Reader` role
- **Reservation portfolio read** — remove Step 4's `Reservations Reader` role (**Home → Reservations → Role assignment**)
- **Export file read** — remove Step 6's `Storage Blob Data Reader` role, or delete the exports from the Exports list

To revoke all access at once, disable the `CkAzureTuner` enterprise application in **Microsoft Entra ID → Enterprise applications**. The Cost Management exports (Step 5) are yours — delete them from the Exports list if you no longer want CloudKeeper to read them.

## A8 — Authorising purchases (later)

When CloudKeeper presents a reservation purchase recommendation that you approve, you'll grant a separate service principal — `CkAzureCommit-Purchaser` — the `Reservation Purchaser` role at billing-account or billing-profile scope. This is granted just-in-time at purchase authorization and may be revoked after the transaction completes.

The analysis service principal (`CkAzureTuner`) is read-only at all times and is never used for write operations on your billing account. Only `CkAzureCommit-Purchaser`, granted under your explicit per-deal authorization, can spend on your invoice.

Procedure for this step will be provided separately at the time of your first purchase.
