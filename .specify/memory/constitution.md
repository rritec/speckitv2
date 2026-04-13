<!-- 
Sync Impact Report
==================
Version: 1.4.0 (Minor Update - Audit Lakehouse & Logging)
Ratification: 2026-04-13
Last Amended: 2026-04-13

Project: rritec Microsoft Fabric Data Pipeline

Principles Established:
  1. Medallion Architecture (Landing → Raw → Integration → Consumption)
  2. Data Quality & Audit Standards
  3. Daily Incremental Processing
  4. Surrogate Key Identity Pattern
  5. Cross-Lakehouse Orchestration

Added Sections:
  - Data Ingestion Standards
  - Transformation & Integration Standards
  - Warehouse Consumption Standards
  - Landing Layer File Backup (Post-Raw Layer Creation)
  - Workspace & Environment Structure
  - Audit & Logging Standards **NEW**

Amendments (v1.3.0 → v1.4.0):
  - Added dedicated Audit Lakehouse: lh_audit
  - Defined audit tables: pipeline_execution_log, data_quality_metrics, lineage_tracking, schema_changes, backup_execution_log
  - Set retention policy: 3 years (1095 days) for all audit records
  - Access control: Read-only for users, append-only for service principals, no delete operations
  - Automatic quarterly purge of records older than 3 years
  - Added AuditLogger.py notebook for audit logging to lh_audit
  - Updated workspace structure to include lh_audit in all environments (Dev, Test, Prod)

Templates Requiring Attention:
  - spec-template.md: ✅ Scope verification complete
  - tasks-template.md: ✅ Task categorization aligned (audit table creation tasks)
  - plan-template.md: ✅ Architecture check complete
-->

# Microsoft Fabric Data Pipeline Constitution

## Core Principles

### I. Medallion Architecture – Mandatory Layering
Every data asset flows through a four-layer medallion architecture: Landing (raw ingest) → Raw (all-data persistence) → Integration (business transformations) → Consumption (presentation-ready, keys applied). No data bypasses or shortcuts between layers are permitted. All lakehouses (Landing/Raw, Integration) and Warehouse (final consumption target) are co-located within the same Microsoft Fabric workspace. Three separate workspaces are maintained for environment isolation: Dev, Test, and Prod. Each environment mirrors the complete medallion architecture with independent data and transformation pipelines. 

### II. Daily Batch Overwrite – Idempotent Loading
Raw and Integration layers MUST implement daily full refresh using overwrite mode. This ensures idempotency: no duplicates, no incremental debt, repeatable results. Any failure or re-run produces identical output. Landing layer accepts new daily comma-delimited files from GitHub; processing is scheduled daily with no manual intervention required.

### III. Audit Column Rigor (NON-NEGOTIABLE)
Every table in Raw and Integration layers MUST include audit columns: `LoadDateTime` (UTC timestamp of load), `SourceFileDate` (originating batch date), `DataQualityFlag` (pass/fail marker), `ProcessingStatus` (success/warning/error). These enable root-cause analysis, lineage tracing, and compliance auditing. Audit columns are immutable after insertion.

### IV. PySpark-First Transformations
All reading, parsing, and transformation logic in Raw and Integration notebooks MUST use PySpark (distributed processing). Single-machine SQL is not permitted for scale. Transformations are declarative (schema-on-read), logged, and version-controlled. Notebook code is treated as production code: peer-reviewed, tested, documented.

### V. Surrogate Key & Warehouse Identity Standards
Consumption layer (Warehouse) MUST implement Microsoft Fabric Warehouse identity columns as surrogate keys. No business keys as primary identifiers. Identity column configuration is immutable once applied; schema changes require new table versions, not altering existing tables. All dimension and fact tables follow Kimball methodology.

## Data Ingestion Standards

**Workspace & Environment Structure**:
- **Single Workspace per Environment**: Dev workspace, Test workspace, Prod workspace (three total).
- **Co-located Resources**: All lakehouses and warehouse within each workspace are in the same Fabric workspace (no cross-workspace dependencies).
- **Environment Isolation**: Data, pipelines, and user access are isolated by workspace; no shared resources between environments.
- **Naming Convention**: 
  - **Workspaces**: `ws_rritec_dev`, `ws_rritec_test`, `ws_rritec_prod`
  - **Lakehouses**: 
    - `lh_rl` – Raw Layer Lakehouse
    - `lh_il` – Integration Layer Lakehouse
    - `lh_audit` – Audit & Logging Lakehouse
  - **Warehouse**: `wh_cl` – Warehouse Consumption Layer
  - **Example Full Names**: 
    - Dev: `ws_rritec_dev` workspace contains `lh_rl`, `lh_il`, `lh_audit`, and `wh_cl`
    - Test: `ws_rritec_test` workspace contains `lh_rl`, `lh_il`, `lh_audit`, and `wh_cl`
    - Prod: `ws_rritec_prod` workspace contains `lh_rl`, `lh_il`, `lh_audit`, and `wh_cl`
- **Promotion Path**: Dev → Test → Prod (one-direction only; no backflow to earlier environments).
- **Access Control**: Service principals and users have environment-specific permissions; Dev access does not grant Prod access.

**Landing Layer Responsibilities**:
- Accept comma-delimited files from GitHub daily via Fabric pipeline (scheduled trigger).
- No data transformation; files are stored as-is in OneLake `/Files/data/` folder.
- Filename convention: `{entity}_{YYYY-MM-DD}.csv` (e.g., `orders_2026-04-13.csv`).
- Retention: 90 days (automated purge policy).
- Schema discovery: Inferred from first file; changes require pipeline owner notification.

**Raw Layer Responsibilities**:
- Read CSV from Landing using PySpark `read.csv()` with auto-inferred schema and fault handling.
- Persist all columns as delta tables in Raw lakehouse (same datatype as source, no casting).
- Apply audit columns before write: `LoadDateTime` = current UTC, `SourceFileDate` = inferred from filename.
- Overwrite mode: `mode="overwrite"` (guarantees idempotency).
- No filtering, no null-filling, no derived columns—purely mechanical ingestion.

**Landing Layer File Backup (Post-Raw Layer Creation)**:
- After Raw layer delta tables are successfully created and validated, backup all processed CSV files from Landing layer.
- Backup location: OneLake `/Files/archive/` folder, organized by date (e.g., `/Files/archive/2026-04-13/`).
- Naming convention: Retain original filename + timestamp (e.g., `orders_2026-04-13_processed_20260413T120000Z.csv`).
- Backup trigger: Automatic, follows successful Raw delta table creation and row-count validation.
- Retention: 180 days (separate from Landing layer 90-day retention; archive is immutable backup).
- Failure handling: If backup fails, flag with alert but do not block downstream Integration layer processing.

## Audit & Logging Standards

**Audit Lakehouse (lh_audit) Responsibilities**:
- Dedicated lakehouse for all pipeline audit logs, data quality metrics, lineage tracking, and compliance records.
- **Audit Tables** (stored in delta format within `lh_audit`):
  - `pipeline_execution_log` – Every notebook/pipeline run: start time, end time, status (success/failure/warning), error messages, row counts processed.
  - `data_quality_metrics` – Per table: row counts, null counts, duplicates, DataQualityFlag failures, quality scores by column.
  - `lineage_tracking` – Source entity → Raw table → Integration table → Warehouse table lineage with processing timestamps.
  - `schema_changes` – All schema modifications: timestamp, affected table, old schema, new schema, reason, approver.
  - `backup_execution_log` – Landing file backup job execution: files backed up, backup location, status, retention expiry date.
- **Partition Strategy**: All audit tables partitioned by `AuditDate` (date of audit event).
- **Retention Policy**: 3 years (1095 days) – immutable historical audit trail for compliance, root-cause analysis, and regulatory requirements.
- **Access**: Read-only for most users; insert/append-only for audit logger service principal; no delete/update operations on existing audit records.
- **Automatic Purge**: Automated job removes records older than 3 years (on schedule: quarterly purge on Jan 1, Apr 1, Jul 1, Oct 1, 00:00 UTC).

## Transformation & Integration Standards

**Integration Layer Responsibilities**:
- Read delta tables from Raw lakehouse using PySpark.
- Apply business transformations: filtering, deduplication, column casting, calculations, joins, and hierarchy flattening.
- Add Integration-layer audit columns: `IntegrationLoadDateTime`, `LineageVersion`, `DataQualityScore` (percentage).
- Persist to Integration lakehouse in delta format with overwrite mode to a different lakehouse.
- Schema changes require new table versions (`_v2`, `_v3` suffix) to avoid breaking downstream dependencies.

**Transformation Validation**:
- Row-count parity check: Raw → Integration row count must match or be documented (e.g., deduplication percentage).
- Null propagation rules: Any null in critical columns → mark `DataQualityFlag = false` for that row (do not skip).
- Column completeness: All required columns must be present; missing columns trigger pipeline failure with alert.

## Warehouse Consumption Standards

**Warehouse Tables**:
- Fact tables: Identity column (Fabric Warehouse managed), dimension foreign keys, measures, audit columns, partition on date.
- Dimension tables: Identity column, type-2 SCD structure (EffectiveDate, EndDate, IsCurrent), business keys immutable.
- Staging tables (if temporary ETL required): Same audit column standards, TTL 24 hours.

**Load Pattern**:
- Read from Integration lakehouse delta tables.
- Map Integration columns to Warehouse schema.
- Generate surrogate keys using Warehouse identity on insert.
- Upsert or merge (not raw insert) to preserve SCD integrity.

**Performance & Monitoring**:
- Partitioning strategy: By date (SourceFileDate in Raw/Integration, LoadDate in Warehouse).
- Index strategy: Clustered on surrogate key + date partition; non-clustered on business keys for query optimization.
- Query timeout: 300 seconds maximum for any single refresh operation (alert on breach).

## Development Workflow

**Code Organization**:
- Separate notebooks per layer: `LandingToRaw.py`, `RawToIntegration.py`, `IntegrationToWarehouse.py`.
- Backup utility notebook: `BackupUtils.py` (handles post-Raw landing file archival, compression, timestamp management).
- Audit utility notebook: `AuditLogger.py` (writes pipeline execution logs, data quality metrics, lineage, schema changes to `lh_audit`).
- Centralized utility notebook: `AuditUtils.py` (audit column functions, logging, retry logic, error handling).
- Version control: All notebooks stored in Git; changes via PR with peer review before merge.

**Testing & Validation**:
- Unit tests for transformation logic (PySpark mocking encouraged).
- Integration test: Daily dry-run pipeline on test workspace; must pass before production schedule activation.
- Regression: Any schema or transformation change must include before-and-after row-count validation.

**Deployment**:
- Feature branches for new transformations; main branch = production.
- Tag release versions in Git for traceability (e.g., `v1.2.3-raw-layer-fix`).
- Rollback procedure: Documented; raw/integration overwrite mode enables safe point-in-time recovery.

## Governance

The Constitution supersedes all other development practices and conventions. Amendments require:
1. Written rationale (principle change, technology upgrade, compliance requirement).
2. Approval from data engineering lead.
3. Migration plan for existing pipelines (if breaking change).
4. Git tag: `constitution-amended-vX.Y.Z` with linked PR.

All pipelines and notebooks MUST verify compliance with this Constitution during code review using the [Compliance Checklist](../templates/compliance-checklist.md). Violations of NON-NEGOTIABLE principles (Audit Column Rigor, Test-First Transformations) trigger immediate remediation or pipeline deactivation.
4
Version control: Every pipeline modification is logged with rationale; lineage metadata is preserved in audit columns. Quarterly council reviews ensure ongoing alignment.

**Version**: 1.3.0 | **Ratified**: 2026-04-13 | **Last Amended**: 2026-04-13
