# Feature Specification: Customer Master Data Pipeline

**Feature Branch**: `001-customer-mdm-pipeline`  
**Created**: 2026-04-13  
**Status**: Draft  
**Project**: rritec Fabric Data Pipeline (Self-Learning)  
**Constitution**: v1.5.0

## Overview

Build an end-to-end customer master data management (MDM) pipeline using the rritec medallion architecture. Daily customer CSV files from GitHub are ingested, deduplicated, enriched with business logic, and loaded into a Type-2 SCD dimension for BI consumption. This feature demonstrates all four layers (Landing → Raw → Integration → Consumption) and core constitution practices: audit rigor, PySpark transformations, surrogate keys, and data quality gates.

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Daily Customer Ingestion (Priority: P1)

Data engineers need to automatically load daily customer updates from GitHub without manual intervention or duplicate records, ensuring all source data is preserved in the Raw layer for historical analysis.

**Why this priority**: Foundational capability; blocks all downstream transformations and BI reporting.

**Independent Test**: Can be fully tested by validating Raw delta table creation with audit columns, row-count parity with source CSV, and successful backup.

**Acceptance Scenarios**:
1. **Given** a new daily customer CSV file is placed in GitHub Landing folder, **When** the `LandingToRaw.py` notebook runs, **Then** all rows are loaded into Raw delta table with `LoadDateTime`, `SourceFileDate`, `DataQualityFlag`, and `ProcessingStatus` audit columns populated.
2. **Given** a CSV with 1000 customers, **When** Raw table is checked, **Then** row count = 1000 (no data loss), and all columns preserve original data types.
3. **Given** Raw table creation succeeds, **When** backup job runs, **Then** CSV is copied to `/Files/archive/{date}/` with timestamp appended.

---

### User Story 2 - Data Quality & Deduplication (Priority: P1)

Data stewards need to identify and handle duplicate or incomplete customer records, ensuring Integration layer contains only high-quality, deduplicated data with quality flags for downstream analysis.

**Why this priority**: Quality gates are NON-NEGOTIABLE per constitution; directly impacts data reliability.

**Independent Test**: Can be tested by comparing Raw vs. Integration row counts, validating deduplication logic (email-based), and confirming DataQualityFlag accuracy on test sample.

**Acceptance Scenarios**:
1. **Given** Raw table with 3 duplicate records (same email), **When** `RawToIntegration.py` runs, **Then** Integration table contains 1 deduplicated record; duplicates logged with `DataQualityFlag = false` and `ProcessingStatus = "warning"`.
2. **Given** a customer record with null email, **When** processed, **Then** row marked with `DataQualityFlag = false` (not filtered out); null count logged to audit.
3. **Given** 100 raw customers, **When** Integration loads, **Then** row-count parity check confirms ≥95% mapped (any >5% variance triggers alert).

---

### User Story 3 - Customer Dimension Creation (Priority: P1)

BI analysts need a Type-2 SCD dimension table in the Warehouse to track customer status changes over time, supporting historical analysis ("How many customers were inactive on a specific date?").

**Why this priority**: Enables BI reporting; demonstrates Warehouse identity column & SCD pattern.

**Independent Test**: Can be tested by inserting test customers with status changes over multiple days, validating surrogate keys, EffectiveDate/EndDate/IsCurrent flags, and historical query accuracy.

**Acceptance Scenarios**:
1. **Given** a customer inserted on day 1 with status "Active", **When** `IntegrationToWarehouse.py` runs, **Then** Warehouse DimCustomer receives surrogate key (identity), EffectiveDate = load date, EndDate = null, IsCurrent = 1.
2. **Given** same customer status changes to "Inactive" on day 2, **When** next load runs, **Then** previous row EndDate updated to day 2 (IsCurrent = 0), new row inserted with EffectiveDate = day 2, IsCurrent = 1.
3. **Given** historical query date = day 1, **When** queried with SCD logic, **Then** returns "Active" status (correct historical state, not current state).

---

### User Story 4 - Audit & Monitoring (Priority: P2)

Operations team needs real-time visibility into pipeline execution, data quality metrics, and failures, enabling rapid troubleshooting and compliance auditing.

**Why this priority**: Supports operations and compliance; enables SLA monitoring.

**Independent Test**: Can be tested by validating pipeline_execution_log entries, data_quality_metrics accuracy, and dashboard display after end-to-end run.

**Acceptance Scenarios**:
1. **Given** a successful pipeline run, **When** audit tables queried, **Then** pipeline_execution_log shows start time, end time, row counts, ProcessingStatus = "success".
2. **Given** data quality issues detected, **When** data_quality_metrics table checked, **Then** null counts, duplicate counts, and quality scores logged per column.
3. **Given** Warehouse dashboard viewed, **When** customer dimension loaded, **Then** shows record count change, quality score %, SCD row count (current vs. historical).

---

## Functional Requirements

### F1: Landing Layer – Daily CSV Ingestion
- Accept daily customer CSV files from GitHub with naming convention `customer_{YYYY-MM-DD}.csv`.
- Schema: CustomerID, FirstName, LastName, Email, Phone, Country, RegistrationDate, Status, LastModifiedDate (inferred from first file).
- Store as-is in OneLake `/Files/data/` folder; no transformation or filtering.
- Automatically detect new files; process daily via scheduled Fabric pipeline.
- Retention: 90 days (automated cleanup).

### F2: Raw Layer – All-Data Persistence with Audit
- Read CSV from Landing using PySpark with schema inference and fault tolerance.
- Persist all columns as delta table in `lh_rl` (Raw Layer Lakehouse) with data types matching source.
- Add audit columns: `LoadDateTime` (UTC), `SourceFileDate` (from filename), `DataQualityFlag` (default true), `ProcessingStatus` (default "success").
- Overwrite mode guarantees idempotency: same CSV loaded twice produces identical Raw table.
- Row-count parity check: log warning if count deviates >20% from previous day.

### F3: Integration Layer – Deduplication & Enrichment
- Read Raw delta table using PySpark.
- **Deduplication Logic**: Group by email; retain latest record by `LastModifiedDate`; mark duplicates with `DataQualityFlag = false`.
- **Enrichment**: Add derived columns: `CustomerSegment` (High/Medium/Low spend logic), `RegistrationYearMonth` (extract from RegistrationDate).
- **Audit Columns**: `IntegrationLoadDateTime`, `LineageVersion = 1`, `DataQualityScore` = percent of rows with `DataQualityFlag = true`.
- Delta table schema: lh_il (Integration Layer Lakehouse) with version suffix if future schema changes needed.
- Overwrite mode for daily refresh.

### F4: Warehouse – Type-2 SCD Dimension
- Read Integration delta table; map to Warehouse schema.
- **Surrogate Key**: Use Warehouse identity column (auto-assigned by Fabric).
- **Type-2 SCD Columns**: `EffectiveDate`, `EndDate` (null = current), `IsCurrent` (1 = current row, 0 = historical).
- **Merge Logic**: On CustomerEmail, if Status/Segment changed, close previous row (set EndDate, IsCurrent = 0) and insert new row (IsCurrent = 1).
- **Partition**: By EffectiveDate for query performance.
- Table: `wh_cl` (Consumption Layer Warehouse) → `DimCustomer` table.

### F5: Audit & Logging
- Write to `lh_audit` delta tables:
  - `pipeline_execution_log`: Row entry per run with start/end time, row counts (Landing, Raw, Integration, Warehouse), status, duration.
  - `data_quality_metrics`: Per-run quality scores, null/duplicate counts per column, DataQualityFlag failure count.
  - `lineage_tracking`: Source (GitHub) → Raw → Integration → Warehouse dependency chain.
  - `backup_execution_log`: Backup job status, files archived, retention expiry date.
- Audit tables partitioned by date for query efficiency.
- Access: Read-only for analysts, append-only for logger service principal.

### F6: Error Handling & Recovery
- **Failure Levels**: 
  - CRITICAL (auto-retry 3x): Schema error, missing file, pipeline timeout.
  - WARNING (log + notify): >5% data drop, >20% row-count variance, quality score <80%.
  - INFO (log only): Duplicate count warnings, edge cases.
- **Notifications**: Slack to `#rritec-fabric-pipeline` on CRITICAL, email digest of WARNINGs.
- **Recovery**: Rerun notebook (overwrite guarantees idempotency); inspect quality metrics before proceeding.

### F7: Testing & Validation
- **Unit Tests**: Test deduplication logic, data enrichment functions (80%+ coverage).
- **Integration Test**: Full pipeline on Test workspace with 7 days of historical customer data.
- **Regression Test**: Pre/post transformation comparison; validate record counts, column distributions.
- **UAT**: Business validation: spot-check 50 customer records in Warehouse; confirm accuracy vs. source system (100% required).

---

## Success Criteria

### Quantitative Metrics
- **Data Completeness**: ≥99% of Landing rows successfully loaded to Raw (no loss).
- **Deduplication Rate**: Identified duplicates = email-based exact matches; Integration row count = unique email count ± <1%.
- **Data Quality Score**: ≥95% of Integration rows have `DataQualityFlag = true`.
- **Pipeline SLA**: 
  - Landing → Raw: <30 minutes
  - Raw → Integration: <60 minutes
  - Integration → Warehouse: <45 minutes
  - Daily completion: Before 05:00 UTC
- **Availability**: Pipeline success rate ≥99% (monthly uptime target).

### Qualitative Metrics
- **Audit Trail**: All pipeline modifications logged in Git; lineage visible in audit tables.
- **Maintainability**: Code peer-reviewed; documented; no hard-coded values.
- **Accuracy**: Warehouse historical queries accurate to 100% vs. manual verification samples.
- **Team Confidence**: Stakeholders comfortable building additional entities on this foundation.

---

## Key Entities

### Raw Layer Table: `tbl_customer_raw`
```
- CustomerID (string, PK)
- FirstName, LastName, Email, Phone, Country (string)
- RegistrationDate, LastModifiedDate (date)
- Status (string: Active/Inactive)
- LoadDateTime (timestamp, NOT NULL)
- SourceFileDate (date, NOT NULL)
- DataQualityFlag (boolean, DEFAULT true)
- ProcessingStatus (string: success/warning/error)
```

### Integration Layer Table: `tbl_customer_integration`
```
- CustomerID, FirstName, LastName, Email, Phone, Country (string)
- RegistrationDate, LastModifiedDate, RegistrationYearMonth (date/string)
- Status, CustomerSegment (string)
- LoadDateTime, SourceFileDate (from Raw)
- DataQualityFlag, ProcessingStatus (from Raw)
- IntegrationLoadDateTime (timestamp)
- LineageVersion (int, DEFAULT 1)
- DataQualityScore (decimal, range 0-100)
```

### Warehouse Table: `DimCustomer` (Type-2 SCD)
```
- CustomerSK (bigint, IDENTITY PRIMARY KEY) [surrogate key]
- CustomerID (string, NOT NULL) [business key]
- FirstName, LastName, Email, Phone, Country (string)
- Status, CustomerSegment (string)
- RegistrationDate (date)
- EffectiveDate (date, NOT NULL, ClusteredIndex)
- EndDate (date, NULL = current)
- IsCurrent (tinyint: 0 = historical, 1 = current)
- LoadDateTime, SourceFileDate (audit)
```

---

## Assumptions

1. **CSV Schema Stability**: Source CSV schema remains consistent; new columns require schema change review (constitution amendment).
2. **Daily Scheduling**: Landing layer receives daily customer CSV at ~22:00–23:00 UTC preceding day; pipeline triggers 23:30 UTC.
3. **Email Uniqueness**: Email is the natural key for deduplication (business assumption); duplicates retained with quality flag for investigation.
4. **Transformation Logic**: Status can only toggle between "Active" and "Inactive"; no other states initially (extensible in future).
5. **Historical Requirement**: Business accepts Type-2 SCD (full history) not Type-1 (latest only); storage/query cost accepted.

---

## Dependencies & Constraints

### Dependencies
- **Upstream**: GitHub folder with daily CSV files (external, out-of-scope for this feature).
- **Downstream**: BI tools (Tableau, Power BI) querying DimCustomer for reporting.
- **Infrastructure**: Fabric workspace (`ws_rritec_dev` → `ws_rritec_test` → `ws_rritec_prod`), lakehouses (`lh_rl`, `lh_il`, `lh_audit`), warehouse (`wh_cl`).
- **Utilities**: `AuditUtils.py`, `BackupUtils.py`, `AuditLogger.py` notebooks (required pre-built).

### Constraints
- **CSV File Size**: Assume ≤100MB per file (compression handled during backup).
- **Performance**: Raw/Integration transformations must complete in SLA window (30/60 min); optimize PySpark if necessary.
- **Storage**: 3-year retention for audit data (1095 days × daily records = ~365K records estimated).
- **Concurrency**: No concurrent pipeline runs (daily schedule, sequential layers).

---

## Scope Boundaries

### In Scope
- Landing → Raw → Integration → Warehouse full pipeline.
- Daily batch processing with overwrite mode.
- Type-2 SCD dimension creation.
- Audit logging and quality metrics.
- Error handling and notifications.
- Unit/integration/UAT testing.

### Out of Scope
- Real-time/streaming ingestion (batch only).
- Additional customer entities (Contact, Address, etc.) – future features.
- BI dashboard development – separate BI team responsibility.
- Historical customer data backfill (assumes fresh start).
- Cross-system reconciliation (data validation with external sources).

---

## Questions for Clarification

[None at this stage; constitution and assumptions cover all critical decisions]

---

## Next Steps

1. **Architecture Design** → `/speckit.plan` to create data models, Spark job architecture, notebook flow.
2. **Task Generation** → `/speckit.tasks` to generate implementation checklist.
3. **Notebook Development** → Build `LandingToRaw.py`, `RawToIntegration.py`, `IntegrationToWarehouse.py`.
4. **Test Suite** → Unit tests (pytest), integration test dataset, UAT validation samples.
5. **Deployment** → Dev validation → Test promotion → Prod release (with sign-offs).
