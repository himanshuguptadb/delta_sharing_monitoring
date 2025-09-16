# Delta Sharing Monitoring Dashboard — README

This README documents the **Delta Sharing Monitoring** Lakeview dashboard defined by the JSON you provided. fileciteturn0file0

---

## What this is
An operational dashboard for monitoring **Delta Sharing** activity in Unity Catalog: usage by shares/recipients, token health, access patterns, and recent errors. It is implemented as a Lakeview (`.lvdash.json`) spec you can import into Databricks.

> **Auth terms used in this dashboard**
>
> - **D2D** = recipients authenticated with **`DATABRICKS`** (Databricks-to-Databricks).
> - **D2O** = recipients authenticated with **`TOKEN`** (Databricks-to-Open/External).
>
> These map directly to the `authentication_type` / `recipient_authentication_type` fields in the queries.

---

## Datasets (logical views used by widgets)

1. **`shares`** — `SELECT * FROM system.information_schema.shares`  
2. **`Last 7 Days Usage`** — filtered view over `system.access.audit` for Unity Catalog Delta Sharing actions in the last 7 days, with derived fields:
   - `errors = regexp_extract(response.error_message, r":(.*?):", 1)`
   - `status_code = response.status_code`
   - `share_name`, `recipient_name`, `share_type = request_params.recipient_authentication_type`
   - `is_ip_access_denied = request_params.is_ip_access_denied`
3. **`Delta Sharing Usage Data`** — unbounded view of the same `system.access.audit` actions (no 7‑day filter) with identical derived fields.
4. **`recipients`** — enriched recipients from `system.information_schema.recipients` joined to IP allowlists and latest token, adding:
   - `allowed_ip_range` (array)
   - `share_status` (“share assigned” / “share not assigned”)
   - `retrieval_status` (“retrieved” / “not retrieved” from activation URL presence)
   - `token_status` (“expiring soon” / “expired” / “never expire” / “not expiring soon”), computed from `expiration_time`
   - Windowing logic picks the most recent token per `recipient_name`

---

## Pages & Widgets

### 1) **Summary**
Fast KPIs and error/usage pivots.

**Key Counters**
- **Total Shares** — `COUNT(share_name)` from `shares`
- **Total Recipients** — `COUNT(recipient_id)` from `recipients`
- **D2O Recipients** — recipients with `authentication_type = 'TOKEN'`
- **D2D Recipients** — recipients with `authentication_type = 'DATABRICKS'`
- **No IP Access List** — recipients where `allowed_ip_range IS NULL`
- **Not Activated** — recipients with `retrieval_status = 'not retrieved'`
- **Tokens Expired** — `token_status = 'expired'`
- **Tokens Expiring in 7 days** — `token_status = 'expiring soon'`
- **Never Expiring Tokens** — `token_status = 'never expire'`
- **D2O Recipients without Shares** — `authentication_type='TOKEN' AND share_status='share not assigned'`
- **D2D Recipients without Shares** — `authentication_type='DATABRICKS' AND share_status='share not assigned'`

**Pivots (last 7 days; `share_type` filtered)**
- **D2D Errors in last 7 days** — rows=`errors`, cell=`COUNT(account_id)` with `status_code IN (400,403,404,500)` and `share_type='DATABRICKS'`
- **D2D shares accessed in last 7 days** — rows=`share_name`, cell=`COUNT(account_id)` with `share_type='DATABRICKS'`
- **D2D recipients accessed in last 7 days** — rows=`recipient_name`, cell=`COUNT(account_id)` with `share_type='DATABRICKS'`
- **D2O shares accessed in last 7 days** — rows=`share_name`, cell=`COUNT(account_id)` with `share_type='TOKEN'`
- **D2O recipients accessed in last 7 days** — rows=`recipient_name`, cell=`COUNT(account_id)` with `share_type='TOKEN'`
- **D2O Errors in last 7 days** — rows=`errors`, cell=`COUNT(account_id)` with `status_code IN (400,403,404,500)` and `share_type='TOKEN'`

### 2) **Usage Detail**
A detail **table** from `Delta Sharing Usage Data` showing per‑event records:
- Columns: `event_date`, `source_ip_address`, `share_name`, `recipient_name`, `share_type`, `status_code`, `errors`  
  (plus many hidden technical fields like `event_time`, `request_id`, `response`, … for drill‑through)

**Page Filters**
- **Share Name** (single select)
- **Recipient Name** (single select)
- **Source IP** (single select)
- **Share Type** (single select: `DATABRICKS` or `TOKEN`)
- **Date** (range picker over `event_date`)
- **Errors** (single select over extracted `errors` string)

### 3) **Recipient Details**
A **table** over `recipients` showing:
- `authentication_type`, `token_status`, `recipient_name`, `recipient_owner`, `created`, `allowed_ip_range`, `share_status`, `retrieval_status`  
  (with additional hidden fields like `region`, `last_altered`, etc.)

---

## Prerequisites & Permissions

- Unity Catalog must emit **audit logs** to `system.access.audit` for Delta Sharing actions.
- Access to `system.information_schema` views:
  - `shares`, `recipients`, `recipient_allowed_ip_ranges`, `recipient_tokens`, `share_recipient_privileges`.
- Warehouse or SQL endpoint with permission to read the above system tables/views.
- The `REGEXP_EXTRACT` and window functions used in the queries must be supported in your SQL runtime.

> If KPIs render as zero/blank, verify your warehouse has access to the `system` schema and that audit logging includes the Delta Sharing actions used here.

---

## Importing the Dashboard

1. In Databricks, open **Lakeview** → **Dashboards**.
2. Click **Import** (or **Create → Import JSON**), then choose the provided `.lvdash.json` file.
3. Select your **SQL Warehouse** and **default catalog/schema** if prompted.
4. Click **Run** to validate queries. Fix any permission or object‑name mismatches that surface.

---

## Customization Tips

- **Time windows:** Adjust the `event_date` filter or hard‑code a different lookback in `Last 7 Days Usage` if desired.
- **Error taxonomy:** Expand `status_code` list or refine the `errors` extraction regex.
- **D2D vs D2O labeling:** Rename widget titles to your organization’s terminology.
- **IP policy:** Flip **“No IP Access List”** to flag *presence* instead by testing `allowed_ip_range IS NOT NULL`.
- **Activation SLAs:** Add a KPI for tokens not retrieved within _N_ days of creation.
- **Data Governance:** Add filters for `recipient_owner` or `region` to segment operational views.

---

## Troubleshooting

- **No data in Usage tables:** Ensure `service_name='unityCatalog'` and Delta Sharing actions are present in `system.access.audit` for your workspace(s).
- **Recipients show “share not assigned”:** This is based on `share_recipient_privileges`. Confirm grants are up to date.
- **Token status seems wrong:** Status derives from the *latest* token per recipient (ROW_NUMBER window). Check for stale tokens.
- **403 / IP denied:** See `is_ip_access_denied` and `allowed_ip_range` to confirm CIDR allowlists.

---

## Field Reference (selected)

- **share_type / authentication_type** — `'DATABRICKS'` or `'TOKEN'`.
- **token_status** — computed from token `expiration_time` relative to `current_timestamp()`.
- **retrieval_status** — `'not retrieved'` if `activation_url` present; otherwise `'retrieved'`.
- **errors** — substring extracted from `response.error_message` via `REGEXP_EXTRACT`.
- **status_code** — HTTP‑like code from `response.status_code`.

---

## Security & Privacy

- Audit rows may include **source IPs** and identity metadata. Treat the dashboard as **sensitive**.
- Limit access via workspace permissions and restrict underlying tables/views as needed.
- If exporting, ensure datasets are anonymized where appropriate.

---

## Versioning

This README reflects the widgets and queries defined in the uploaded Lakeview spec. If you modify the JSON, update this README accordingly.
