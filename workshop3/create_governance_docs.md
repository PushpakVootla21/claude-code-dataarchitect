# Plan: Governance Deliverables (`data_lineage.md`, `governance_handover.md`)

Scope: review the actual project (all 4 workshop folders, scripts, docs) and
the live `DataWarehouse36` database (via `sqlconection` in `.env`) to produce
two evidence-based deliverables. Everything below comes from files read and
queries run in this session, not assumptions. **Plan only - the two
deliverable files are written in a separate, later step.**

## Evidence gathered (basis for both deliverables)

**Project files reviewed:** every `.md` in the repo, `ddl_bronze.sql`,
`loadbronze.sql`, `blobconnector.sql`, `schemacreate.sql`,
`kafka-stream-copy/*.js`, `pipeline/yaml/*.yaml`, `pipeline/run.ts`,
`workshop3/report/*` (Prisma schema + queries), `workshop4/cleartables.sql`,
`CLAUDE.md`, git history/remote, `.gitignore` files.

**Database queried live (schema/security metadata only, no data dump):**
`INFORMATION_SCHEMA.TABLES`/`COLUMNS`, `sys.foreign_keys`,
`sys.masked_columns`, `sys.sensitivity_classifications`,
`sys.security_policies`, `sys.database_audit_specifications`,
`sys.extended_properties`, `sys.database_principals`,
`sys.database_role_members`, `sys.external_data_sources`,
`sys.database_scoped_credentials`, `sys.dm_database_encryption_keys`.

**Key facts that will drive both documents:**

1. **18 real tables** across `bronze` (7), `silver` (7), `gold` (4). No
   views exist yet (`cleartables.sql` is written to also handle views,
   suggesting views were anticipated but none exist today - a gap between
   tooling and reality worth noting, not a bug).
2. **Zero governance controls exist in the database**: `sys.masked_columns`,
   `sys.sensitivity_classifications`, `sys.security_policies`,
   `sys.database_audit_specifications`, and `sys.extended_properties` (which
   would hold column descriptions/a data dictionary) all returned 0 rows.
   Only one non-system database user (`dbo`, `db_owner`) - no separate
   ingestion/BI roles exist despite `workshop1/security_handover.md`
   recommending them.
3. **TDE is enabled** (`encryption_state = 3`) - the one security control
   from the handover that was already true by default, as the handover
   itself predicted.
4. **`workshop1/blobconnector.sql` still contains the hardcoded master key
   password and a live SAS token in plain text**, and is tracked in git
   (`git ls-files` confirms it; `git log` shows it went in on "first
   commit"). The repo has a real GitHub remote
   (`github.com/karthikeyanVK/claude-code-dataarchitect`). The handover's
   **P0 ("rotate before Workshop 2 starts")** was never carried out - this
   is the single highest-severity open item, unchanged since Workshop 1.
5. **Silver did partially address P1**: gender and country codes are
   normalized (`silver.crm_cust_info`, `silver.erp_cust_az12`,
   `silver.erp_loc_a101` - verified via the Part 3 checks, all passing) and
   invalid birthdates are nulled (`bdate_not_future` check, 0 violations).
   **But name masking/tokenization was never implemented** -
   `cst_firstname`/`cst_lastname` flow through Silver (trimmed only) and
   into `gold.gold_dim_customer.first_name`/`last_name` as plain text, and
   are directly selectable via the Workshop 3 Prisma schema (not currently
   rendered on either dashboard page, but queryable). P1 is half-done, not
   done.
6. **P2 (RLS/CLS on Gold, audit logging) was never implemented** -
   confirmed by the empty `sys.security_policies` and
   `sys.database_audit_specifications` results above. The re-identification
   risk the handover described (name + birthdate + gender + country joined
   in `gold.gold_dim_customer`) is live and unmitigated today.
7. **No data dictionary exists anywhere** (no extended properties, no
   dedicated doc) except informally: `pipeline/yaml/*.yaml`'s
   `table_details.columns` blocks (written during Workshop 2) document
   column-level source/type/transformation for every Bronze->Silver and
   Silver->Gold table - this is real, usable lineage/dictionary material,
   just not consolidated or in dictionary form.
8. **Two other hygiene gaps found, unrelated to security but real:**
   `workshop4/readme.md` is a genuine 0-byte file (violates this project's
   own `CLAUDE.md` rule against empty files). `workshop/workshop.sqlproj` is
   an unused, empty SQL Database Project scaffold (no `.sql` files inside
   it) sitting alongside the real, actively-used pipeline scripts.
9. **`security_handover.md` (in both `workshop1/` and `workshop2/`) is not
   version-controlled** - `git status` shows both as untracked. The
   governance-relevant handoff artifact between Workshop 1 and 2 currently
   exists only on this local disk.

## `data_lineage.md` - planned structure

1. **Lineage table**, one row per dataset, source through Gold:

   | Source | Bronze | Silver | Gold | Loaded by |
   |---|---|---|---|---|
   | `source_crm/cust_info.csv` (Blob) | `bronze.crm_cust_info` | `silver.crm_cust_info` | `gold.gold_dim_customer` | `loadbronze.sql` (`bronze.load_bronze` proc, `BULK INSERT`) -> `pipeline/run.ts` (`crm_cust_info.yaml`) -> `pipeline/run.ts` (`gold_dim_customer.yaml`) |
   | `source_crm/prd_info.csv` (Blob) | `bronze.crm_prd_info` | `silver.crm_prd_info` | `gold.gold_dim_product` | same pattern (`crm_prd_info.yaml`, `gold_dim_product.yaml`) |
   | `source_crm/sales_details.csv` (Blob) | `bronze.crm_sales_details` | `silver.crm_sales_details` | `gold.gold_fact_sales` | same pattern |
   | `source_erp/CUST_AZ12.csv` (Blob) | `bronze.erp_cust_az12` | `silver.erp_cust_az12` | `gold.gold_dim_customer` (joined in) | same pattern |
   | `source_erp/LOC_A101.csv` (Blob) | `bronze.erp_loc_a101` | `silver.erp_loc_a101` | `gold.gold_dim_customer` (joined in) | same pattern |
   | `source_erp/PX_CAT_G1V2.csv` (Blob) | `bronze.erp_px_cat_g1v2` | `silver.erp_px_cat_g1v2` | `gold.gold_dim_product` (joined in) | same pattern |
   | Azure Event Hubs `campaign-events` topic | `bronze.mkt_campaign_events` | `silver.mkt_campaign_events` | `gold.gold_dim_campaign` + `gold.gold_fact_sales` (attribution join) | `kafka-stream-copy/consume_campaign_events.js` -> `pipeline/run.ts` (`mkt_campaign_events.yaml`, `gold_dim_campaign.yaml`, `gold_fact_sales.yaml`) |

2. **Mermaid flowcharts:**
   - One end-to-end flow: CSV/Blob + Kafka -> Bronze -> Silver -> Gold ->
     `workshop3/report` (Prisma/Next.js).
   - One per-layer detail diagram for the Bronze->Silver->Gold transform
     graph (matches the actual FK/join graph verified in Workshop 2:
     `crm_cust_info` is the hub every other customer-related table joins
     against; `gold_fact_sales` is the sink joining all three dimensions).

3. **"Lineage tracking today" section**: honest gap statement - lineage is
   currently *reconstructable* from `pipeline/yaml/*.yaml` (each file
   states its own `source`/`target`) and this new document, but there is no
   automated/tool-based lineage tracking (no OpenLineage, no Purview scan,
   no column-level lineage graph). Recorded as a gap, not glossed over.

## `governance_handover.md` - planned structure

1. **Metadata / data dictionary**: one consolidated table per layer
   (Bronze/Silver/Gold), every real table and column, type, source, and a
   `classification` column populated only where the evidence supports it:
   - `Direct PII`: `first_name`/`last_name` (`gold_dim_customer`),
     `cst_firstname`/`cst_lastname` (Silver/Bronze CRM).
   - `Quasi-identifier`: `birthdate`, `gender`, `country`,
     `marital_status` (re-identifying in combination, per
     `security_handover.md`'s own finding, verified still true in Gold).
   - `Commercially sensitive`: `sales_amount`, `price`, `cost`.
   - `Not classified / no evidence of formal classification` for
     everything else - explicitly marked as a gap (echoing finding #7 above:
     zero rows in `sys.sensitivity_classifications`), not left blank
     silently.
   - **Owner**: recorded as "Unknown - no owner metadata exists in the
     project or database" for every table. The `.env` files' comment
     (`Auto-populated from az login - foxpr@hotmail.com`) identifies who
     provisioned the *infrastructure/subscription*, not a data/table owner,
     and will be cited as exactly that, not conflated with data ownership.

2. **Lineage summary**: references `data_lineage.md`, doesn't duplicate it.

3. **Architecture diagrams** (Mermaid): the medallion layer diagram from
   `data_lineage.md`, plus a security-overlay version marking where PII
   currently sits unmitigated (Bronze CRM/ERP tables, Silver customer
   tables, `gold.gold_dim_customer`) versus where no PII exists (product,
   campaign, sales-amount data).

4. **Governance handover findings** (the real substance, all evidence-based
   per above): P0 credential exposure still open and now on a public git
   remote; P1 half-done (normalization yes, masking no); P2 fully open (no
   RLS/CLS/audit); no data dictionary previously existed; two hygiene gaps
   (empty `workshop4/readme.md`, unused `workshop/workshop.sqlproj`);
   `security_handover.md` files uncommitted.

5. **Risks**, carried forward and updated from `security_handover.md`'s
   "Risk by Layer" section, annotated with **current status** (open/
   partially mitigated/mitigated) instead of restating them as still
   hypothetical.

6. **Evidence-based recommendations**, prioritized (reusing and updating
   the original P0/P1/P2 framing rather than inventing a new scheme):
   - **P0 (unchanged, still not done)**: rotate the SAS token and master
     key password in `blobconnector.sql`; remove the hardcoded values from
     git history if the remote is or will be shared.
   - **P1 (partially done)**: mask or tokenize `first_name`/`last_name`
     before Gold; commit `security_handover.md` in both folders so the
     handoff artifact itself isn't at risk of loss.
   - **P2 (not started)**: add RLS or a column-level security view on
     `gold.gold_dim_customer`; enable SQL Audit; apply SQL Server Data
     Classification labels (would populate the currently-empty
     `sys.sensitivity_classifications`) so classification stops being a
     manual doc and becomes queryable metadata.
   - **Hygiene (low severity, real)**: fill or remove
     `workshop4/readme.md`; remove or populate `workshop/workshop.sqlproj`.

7. **Open questions**: carried forward from `security_handover.md` (owner
   of Gold customer dimension, data residency requirement, Bronze retention
   policy) since none were answered by anything found in this review -
   repeating them here as still-open, not re-inventing new ones.

## What's explicitly NOT done (per the prompt's own constraints)

- No remediation is implemented in this step (no credential rotation, no
  masking, no RLS) - this plan and the two resulting documents are
  reporting/handover artifacts only.
- No classifications, owners, or lineage edges are invented where no
  evidence exists - every such gap is recorded as "Unknown" / "gap" in the
  actual deliverables, per the prompt's explicit instruction.
