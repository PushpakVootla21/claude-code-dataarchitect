# Security Handover: Workshop 1 to Workshop 2

Data Security Architect review of `workshop1/datasets/` (CRM and ERP source files) and the connection setup scripts, prior to Bronze to Silver to Gold movement.

## Scope Reviewed

- `source_crm/cust_info.csv` (18,493 rows): customer identity and demographics
- `source_crm/prd_info.csv` (397 rows): product catalog
- `source_crm/sales_details.csv` (60,398 rows): order transactions
- `source_erp/CUST_AZ12.csv` (18,483 rows): customer birthdate and gender
- `source_erp/LOC_A101.csv` (18,484 rows): customer country
- `source_erp/PX_CAT_G1V2.csv` (36 rows): product category reference
- `blobconnector.sql`, `schemacreate.sql`: connection and schema setup scripts

## Key Findings

### 1. PII present in source data

- `cust_info.csv` holds direct identifiers: first name, last name, marital status, gender, tied to `cst_id` / `cst_key`.
- `CUST_AZ12.csv` holds birthdate and gender, joinable to `cust_info.csv` via the customer key embedded in `CID` (e.g. `NASAW00011000` maps to `AW00011000`).
- `LOC_A101.csv` holds country of residence, joinable the same way via `CID` (e.g. `AW-00011000`).
- Joined across all three, a record becomes name, birthdate, gender, marital status, and country, enough to re-identify an individual. No single file is sensitive alone; the join across CRM and ERP is what creates the risk.
- No email, phone, government ID, or payment card data was found in any file.

### 2. Exposed credentials committed to source control

- `workshop1/blobconnector.sql` (tracked in git) contains a hardcoded master key password (`ClaudeCodeMasterKey123!`) and a live Blob Storage SAS token (full query string with `sig=`) in plain text. Anyone with read access to the repository can read and reuse both.
- This is distinct from `.env`, which is correctly gitignored. The SQL script bypasses that protection by embedding the secret directly.

### 3. Data quality issues that will undermine downstream controls

- `GEN` (ERP) and `cst_gndr` (CRM) use inconsistent codings: blank, single space, `M`/`F`, `Male`/`Female`, with trailing spaces. Any masking or access rule keyed on gender will silently miss rows unless normalized first.
- `CNTRY` uses inconsistent codings for the same country: `DE`/`Germany`, `US`/`USA`/`United States`, plus blank and whitespace-only values. Any residency-based access control keyed on country will be unreliable until standardized.
- `BDATE` contains sentinel/invalid future dates (e.g. `9999-09-11`, `9999-11-20`), alongside plausible ages back to 1916. These must not reach Gold or BI as real birthdates.
- `cust_info.csv` has blank first name (8 rows), last name (7 rows), marital status (7 rows), and gender (4,578 rows, roughly 25%).

### 4. Sensitive but non-personal data

- `sales_details.csv` contains order-level revenue (`sls_sales`, `sls_price`), which is commercially sensitive (competitor or partner inference risk) though not personal data.

## Recommended Controls

1. **Rotate and remove the exposed credentials.** Rotate the SAS token and master key password referenced in `blobconnector.sql` immediately, since they are visible in git history and cannot be un-committed by editing the file alone. Replace the hardcoded values with a reference to `.env` (or a Key Vault backed scoped credential) and add `blobconnector.sql`'s secret line to history-scrubbing if the repo will be shared further.
2. **Mask or pseudonymize direct identifiers before Silver.** `cst_firstname`, `cst_lastname` should be masked or tokenized in any layer beyond Bronze that is broadly readable. Keep a restricted-access mapping table only where a legitimate business need exists.
3. **Restrict the CRM+ERP join.** The combination of name, birthdate, gender, and country should only be queryable by roles with an explicit need, enforced with row-level security (RLS) or a column-level security (CLS) view rather than open access to the joined Gold table.
4. **Normalize categorical PII fields before they're used as access-control keys.** Standardize gender and country codes in Silver so masking/RLS rules keyed on these fields behave consistently.
5. **Null out or quarantine sentinel dates.** `BDATE` values like `9999-09-11` should be set to null (not passed through) in Silver, with the row flagged for the data quality report rather than silently corrected.
6. **Encrypt at rest and in transit.** Confirm Azure SQL Transparent Data Encryption (TDE) is enabled (default for Azure SQL) and that `Encrypt=true` stays enforced in all connection strings, which the current `.env` already does.
7. **Add audit logging on Gold and BI access.** Enable SQL Audit or diagnostic logging on the database so access to the joined customer view is traceable, since this is the layer with re-identification risk.

## Risk by Layer

- **Bronze**: Raw PII lands as-is; access must be restricted to the ingestion/engineering role only, not BI consumers.
- **Silver**: Cleansing must normalize gender/country and null invalid birthdates before this layer is exposed to any broader audience; this is also where masking/tokenization of names should be applied.
- **Gold**: The star schema will likely join customer dimensions across CRM and ERP, recreating the re-identification risk described above; this layer needs RLS/CLS, not just cleansing.
- **BI**: Reports and dashboards must consume only masked/aggregated fields; raw name, birthdate, and gender should not be selectable in BI tooling unless the consumer's role is explicitly authorized.

## Priorities

- **P0 (before Workshop 2 starts)**: Rotate the SAS token and master key password committed in `blobconnector.sql`.
- **P1 (during Silver layer build, Workshop 2 Part 3)**: Normalize gender/country codes, null invalid birthdates, mask/tokenize customer names.
- **P2 (during Gold/reporting build, Workshop 2 Part 4 and Workshop 3)**: Apply RLS/CLS on the joined customer dimension, enable audit logging on Gold and BI access paths.

## Open Questions

- Who is the intended audience for the Gold-layer customer dimension: is column-level masking sufficient, or does compliance require full tokenization with a separate re-identification service?
- Is there a data residency requirement tied to the `CNTRY` field (e.g. EU customers) that constrains where Gold/BI data can be stored or processed?
- What is the retention policy for raw Bronze data once Silver/Gold are populated, given it holds unmasked PII?
