<!-- 
Sync Impact Report
==================
Version: 1.5.0 (Minor Update - Quality, Error Handling, Testing Standards)
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
  - Audit & Logging Standards
  - Data Quality Thresholds & Validation Standards **NEW**
  - Error Handling & Alerting Standards **NEW**
  - Testing Standards & Quality Gates **NEW**

Amendments (v1.4.0 → v1.5.0):
  - Added Data Quality Thresholds: null tolerance (5%), quality scores (80%), SLA targets (30/60/45 min layers, 05:00 UTC daily)
  - Added Error Handling: failure levels (CRITICAL/WARNING/INFO), retry logic (3x exponential backoff), notification channels (Slack/Email/Dashboard)
  - Added Testing Standards: unit test coverage (80%+), integration tests (mandatory for new entities), regression testing (pre/post validation)
  - Added Promotion Workflow: Dev → Test → Prod with code review, integration testing, UAT sign-off
  - Added Recovery Procedures: layer-specific rollback steps, idempotent re-execution guarantees
  - Added SLA Targets: overall pipeline completion by 05:00 UTC, data freshness ≤ 24 hours

Templates Requiring Attention:
  - spec-template.md: ✅ Scope verification complete
  - tasks-template.md: ✅ Task categorization aligned (quality gates, testing tasks)
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

## Data Quality Thresholds & Validation Standards

**Quality Gate Rules (NON-NEGOTIABLE)**:
- **Raw Layer**:
  - Null tolerance: ≤ 5% nulls per column → mark `DataQualityFlag = false` for affected rows (do not filter).
  - Duplicate detection: No business-key duplicates allowed; exact duplicates must be logged with `ProcessingStatus = "warning"`.
  - Schema compliance: All expected columns present; missing columns → `ProcessingStatus = "error"`, pipeline fails with alert.
  - Row-count parity: Previous day row count deviation > 20% (up or down) → trigger investigation alert (non-blocking).

- **Integration Layer**:
  - Data completeness: ≥ 95% of source rows must map to Integration tables (transformation drop-off > 5% triggers alert).
  - Column quality: `DataQualityScore` ≥ 80% per column; <80% → marked as `LineageVersion._validation_needed`.
  - Referential integrity: Foreign keys to dimension tables must exist (validate before insert); missing → quarantine row with alert.
  - Business logic validation: Custom rules per entity (e.g., `OrderAmount > 0`, `OrderDate <= ProcessDate`); failures logged to `data_quality_metrics`.

- **Warehouse Consumption Layer**:
  - Identity column uniqueness: Surrogate keys must be unique (no duplicates); primary key violation → rollback transaction.
  - Fact table grain: No duplicate grain keys (combination of dimension keys + measure keys); duplicates → alert and investigation.
  - SCD Type-2 integrity: No overlapping effective date ranges; `IsCurrent` flag correct; violations → quarantine & alert.

**SLA Targets**:
- Landing → Raw transformation: Must complete within **30 minutes** of file arrival (alert if exceeded).
- Raw → Integration transformation: Must complete within **60 minutes** (alert if exceeded).
- Integration → Warehouse load: Must complete within **45 minutes** (alert if exceeded).
- Overall pipeline SLA: Must complete daily cycle by **05:00 UTC** (alert if exceeded).
- Data freshness: Latest data in Warehouse must be ≤ 24 hours old (alert if stale).

## Error Handling & Alerting Standards

**Failure Detection & Response**:
- **Pipeline Failure Levels**:
  - **CRITICAL** (automatic retry + notify lead): Schema errors, identity key violation, file not found, pipeline timeout.
  - **WARNING** (log + notify team): 5-20% data drop-off, any DataQualityFlag failures, SCD integrity warnings.
  - **INFO** (log only): Null tolerance at edge cases, minor deviations in row counts.

- **Retry Logic**:
  - CRITICAL failures: Automatic retry up to 3 times (exponential backoff: 5s, 10s, 30s).
  - If retries exhaust: Escalate to alert (see notification channels below).
  - Manual intervention required if retry count exceeds 3.

- **Notification Channels**:
  - **Slack** (immediate): All CRITICAL failures, SLA breaches → channel `#rritec-fabric-pipeline`
  - **Email** (hourly digest): WARNING-level issues, quality threshold violations → team@rritec.com
  - **Dashboard** (real-time): Custom Warehouse dashboard tracking pipeline health, failure counts, quality metrics.
  - **Escalation**: If CRITICAL unresolved > 1 hour → page on-call data engineer.

- **Error Logging**:
  - All errors logged to `pipeline_execution_log` with stack trace, affected rows, recovery action taken.
  - Error context includes: environment (Dev/Test/Prod), layer (Landing/Raw/Integration/Warehouse), entity name, timestamp, user/SP identity.
  - Errors queryable via Warehouse dashboard for troubleshooting & root cause analysis.

**Recovery Procedures**:
- **Raw Layer Failure**: Rerun `LandingToRaw.py`; overwrite mode guarantees idempotent re-execution (no duplicates).
- **Integration Layer Failure**: Rerun `RawToIntegration.py`; inspect quality metrics before retry.
- **Warehouse Load Failure**: Validate SCD & key integrity; rollback transaction; fix schema/constraints; rerun `IntegrationToWarehouse.py`.
- **Backup Failure**: Alert but do not block downstream (backup retry scheduled next 6 hours; manual intervention if 3 failures).

## Testing Standards & Quality Gates

**Test Coverage Requirements (MANDATORY)**:
- **Unit Tests**: ≥ 80% code coverage for all transformation logic (PySpark functions, business rules).
  - Test frameworks: pytest (Python), with Spark mock fixtures.
  - Test file location: `tests/unit/test_{notebook_name}.py` in Git repository.
  - Run before PR submission: `pytest tests/unit/` must pass 100%.

- **Integration Tests**: Mandatory for all new entities and schema changes.
  - Dry-run on Test workspace: Full pipeline execution with small dataset (last 7 days).
  - Validate: Row counts, quality metrics, SCD integrity, audit logs written correctly.
  - Test file location: `.specify/templates/integration-test-plan.md` for each entity.
  - Required frequency: Daily (before production schedule activation).

- **Regression Tests**: Any transformation change requires before-and-after validation.
  - Compare: Row counts, column distributions, null patterns, quality scores pre/post change.
  - Test data: Production sample from previous day; run new code on same data; confirm no data loss/corruption.
  - Sign-off: Data engineering lead approves regression results before Prod deployment.

**Promotion Workflow (Dev → Test → Prod)**:
1. Developer: Git commit on feature branch, unit tests pass (80%+ coverage).
2. Code Review: Data engineering lead reviews transformations, audit logging, error handling.
3. Test Promotion: Merge to test branch → automatic run on Test workspace.
4. Integration Test: Full pipeline runs; quality gates validated; manual sign-off required.
5. Prod Promotion: Tag release version in Git (`v1.x.x`); merge to main branch.
6. Prod Execution: Scheduled pipeline runs; monitoring active; rollback plan documented.
7. Post-Deploy Validation: Verify Prod data matches expected quality metrics within 1 hour of load.

**UAT Sign-Off**:
- Business stakeholders required to validate sample records in Warehouse (min 50 rows per entity).
- Spot-check: Compare 5 random records Warehouse vs. source system; 100% accuracy required.
- Quality sign-off: Data quality metrics acceptable; no critical issues blocking production release.
- Formal approval: Email sign-off from business data owner before final Prod deployment.

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

**Version**: 1.5.0 | **Ratified**: 2026-04-13 | **Last Amended**: 2026-04-13
