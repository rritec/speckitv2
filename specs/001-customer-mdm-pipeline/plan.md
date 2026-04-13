# Implementation Plan: Customer Master Data Pipeline

**Branch**: `001-customer-mdm-pipeline` | **Date**: 2026-04-13 | **Spec**: [001-customer-mdm-pipeline/spec.md](spec.md)

**Note**: Plan created by `/speckit.plan` command executing medallion architecture workflow.

## Summary

Build an end-to-end customer master data management (MDM) pipeline leveraging the rritec Microsoft Fabric medallion architecture. Daily customer CSV files from GitHub are ingested into Landing, loaded to Raw with audit columns, deduplicated and enriched in Integration layer, and finally loaded into a Type-2 SCD dimension in Warehouse. This implementation demonstrates core rritec practices: PySpark transformations, audit rigor, quality gates, and error handling across all four data layers.

---

## Technical Context

**Language/Version**: Python 3.9+, PySpark 3.x  
**Primary Dependencies**: 
- Apache Spark (Delta Lake)
- Microsoft Fabric runtime (default cluster config)
- pandas (for metadata operations)
- pytest + pyspark-testing (unit tests)

**Storage**: Microsoft Fabric OneLake (delta tables)
- lh_rl: Raw Layer Lakehouse
- lh_il: Integration Layer Lakehouse
- lh_audit: Audit & Logging Lakehouse
- wh_cl: Warehouse Consumption Layer

**Testing**: pytest (PySpark SQL mocking frameworks)

**Target Platform**: Microsoft Fabric Notebooks (.py format), Fabric pipelines (scheduled daily triggers)

**Project Type**: Data pipeline / ETL / Data warehouse

**Performance Goals**: 
- Landing → Raw: <30 minutes
- Raw → Integration: <60 minutes
- Integration → Warehouse: <45 minutes
- Daily completion: Before 05:00 UTC

**Constraints**: 
- Batch processing only (no streaming)
- Daily scheduling (no real-time)
- <500MB CSV file size assumed
- Email uniqueness for deduplication

**Scale/Scope**: 
- Initial: 10K–50K customer records daily
- Growth: Support 100K+ records (scalable via partitioning)
- Historical: 3-year audit retention (1M+ records)

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Core Principles Alignment** ✅:

| Principle | Requirement | Status |
|-----------|-------------|--------|
| **I. Medallion Architecture** | Landing → Raw → Integration → Consumption | ✅ Fully applied |
| **II. Daily Batch Overwrite** | Idempotent loading, overwrite mode | ✅ Delta overwrite mode specified |
| **III. Audit Column Rigor** (NON-NEGOTIABLE) | LoadDateTime, SourceFileDate, DataQualityFlag, ProcessingStatus | ✅ All 4 columns on Raw/Integration |
| **IV. PySpark-First** | Distributed processing, no single-machine SQL | ✅ PySpark for all transforms |
| **V. Surrogate Key Standards** | Warehouse identity columns, SCD Type-2 | ✅ Identity column + SCD design confirmed |

**Data Quality Gates** ✅:
- Raw: ≤5% nulls, duplicates logged, schema compliance
- Integration: ≥95% completeness, 80%+ quality score
- Warehouse: Unique surrogate keys, SCD integrity

**Error Handling** ✅:
- CRITICAL/WARNING/INFO levels defined
- Retry logic (3x exponential backoff)
- Notification channels (Slack, email, dashboard)

**Testing Standards** ✅:
- Unit tests ≥80% coverage
- Integration tests mandatory
- Regression testing on schema changes
- UAT sign-off required

**Governance** ✅:
- Feature branch workflow (Git)
- Code review + peer sign-off
- Promotion path: Dev → Test → Prod
- Audit logging in lh_audit (3-year retention)

**GATE RESULT**: ✅ **PASS** – Specification fully aligned with rritec Constitution v1.5.0

---

## Project Structure

### Documentation (this feature)

```
specs/001-customer-mdm-pipeline/
├── plan.md                      # This file (Phase 1-2 output)
├── spec.md                      # Feature specification
├── research.md                  # Phase 0 output (if clarifications needed)
├── data-model.md                # Phase 1 output
├── contracts/                   # Phase 1 output (API/schema definitions)
│   ├── landing-schema.json
│   ├── raw-schema.json
│   ├── integration-schema.json
│   └── warehouse-schema.json
├── quickstart.md                # Phase 1 output (getting started guide)
└── tasks.md                     # Phase 2 output (implementation tasks)
```

### Source Code (Microsoft Fabric Notebooks + Tests)

```
notebooks/
├── LandingToRaw.py              # Landing → Raw transformation
├── RawToIntegration.py          # Raw → Integration transformation
├── IntegrationToWarehouse.py    # Integration → Warehouse load
├── AuditUtils.py                # Shared audit column functions
├── BackupUtils.py               # Landing file backup utilities
└── AuditLogger.py               # Audit table logging

tests/
├── unit/
│   ├── test_landing_to_raw.py
│   ├── test_raw_to_integration.py
│   ├── test_integration_to_warehouse.py
│   └── conftest.py (pytest fixtures)
├── integration/
│   ├── test_end_to_end_pipeline.py
│   └── test_data.csv (7-day sample dataset)
└── uat/
    └── uat-validation-checklist.md
```

---

## Phase 0: Research & Clarification

**Status**: No clarifications needed; specification is complete.

**Key Decisions Already Made** (from spec):
- ✅ CSV schema: CustomerID, FirstName, LastName, Email, Phone, Country, RegistrationDate, Status, LastModifiedDate
- ✅ Deduplication key: Email (latest by LastModifiedDate wins)
- ✅ Enrichment: CustomerSegment (derived field) + RegistrationYearMonth
- ✅ SCD Type: Type-2 (historical tracking)
- ✅ Surrogate key: Warehouse identity column
- ✅ Audit pattern: All four audit columns (LoadDateTime, SourceFileDate, DataQualityFlag, ProcessingStatus)
- ✅ Retention: Landing 90-day, Raw/Integration daily overwrite, Audit 3-year

**Proceeding to Phase 1: Design & Contracts**

---

## Phase 1: Design & Data Models

### 1. Data Model Definitions

**Extracted Entities & Relationships**:

```
Entity: Customer
├── Landing Layer: raw CSV (no transformation)
├── Raw Layer: all columns + audit
├── Integration Layer: deduplicated + enriched
└── Warehouse: SCD Type-2 dimension

Key Attributes:
  - Business Key: Email (uniqueness constraint)
  - Surrogate Key: CustomerSK (Warehouse identity)
  - Versioning: EffectiveDate, EndDate, IsCurrent (SCD Type-2)
  - Audit: LoadDateTime, SourceFileDate, DataQualityFlag, ProcessingStatus
  - Quality: DataQualityScore, LineageVersion
```

### 2. Schema Contracts

**Landing Schema** (CSV from GitHub):
```json
{
  "entity": "customer",
  "format": "csv",
  "filename_pattern": "customer_YYYY-MM-DD.csv",
  "encoding": "UTF-8",
  "delimiter": ",",
  "columns": [
    {"name": "CustomerID", "type": "string", "nullable": false},
    {"name": "FirstName", "type": "string", "nullable": true},
    {"name": "LastName", "type": "string", "nullable": true},
    {"name": "Email", "type": "string", "nullable": false},
    {"name": "Phone", "type": "string", "nullable": true},
    {"name": "Country", "type": "string", "nullable": true},
    {"name": "RegistrationDate", "type": "date", "nullable": true},
    {"name": "Status", "type": "string", "nullable": false, "values": ["Active", "Inactive"]},
    {"name": "LastModifiedDate", "type": "date", "nullable": false}
  ]
}
```

**Raw Layer Schema** (lh_rl):
```json
{
  "table_name": "tbl_customer_raw",
  "storage_format": "delta",
  "location": "lh_rl/Files/tables/customer/raw",
  "partition_by": null,
  "columns": [
    {"name": "CustomerID", "type": "string", "nullable": false},
    {"name": "FirstName", "type": "string", "nullable": true},
    {"name": "LastName", "type": "string", "nullable": true},
    {"name": "Email", "type": "string", "nullable": false},
    {"name": "Phone", "type": "string", "nullable": true},
    {"name": "Country", "type": "string", "nullable": true},
    {"name": "RegistrationDate", "type": "date", "nullable": true},
    {"name": "Status", "type": "string", "nullable": false},
    {"name": "LastModifiedDate", "type": "date", "nullable": false},
    {"name": "LoadDateTime", "type": "timestamp", "nullable": false, "description": "UTC timestamp of load"},
    {"name": "SourceFileDate", "type": "date", "nullable": false, "description": "Date extracted from filename"},
    {"name": "DataQualityFlag", "type": "boolean", "nullable": false, "default": true},
    {"name": "ProcessingStatus", "type": "string", "nullable": false, "default": "success"}
  ],
  "write_mode": "overwrite",
  "retention_days": null
}
```

**Integration Layer Schema** (lh_il):
```json
{
  "table_name": "tbl_customer_integration",
  "storage_format": "delta",
  "location": "lh_il/Files/tables/customer/integration",
  "dedupe_key": ["Email"],
  "dedupe_logic": "Retain latest by LastModifiedDate; mark duplicates with DataQualityFlag=false",
  "columns": [
    {"name": "CustomerID", "type": "string", "nullable": false},
    {"name": "FirstName", "type": "string", "nullable": true},
    {"name": "LastName", "type": "string", "nullable": true},
    {"name": "Email", "type": "string", "nullable": false},
    {"name": "Phone", "type": "string", "nullable": true},
    {"name": "Country", "type": "string", "nullable": true},
    {"name": "RegistrationDate", "type": "date", "nullable": true},
    {"name": "RegistrationYearMonth", "type": "string", "nullable": true, "format": "YYYY-MM", "derived": true},
    {"name": "Status", "type": "string", "nullable": false},
    {"name": "CustomerSegment", "type": "string", "nullable": true, "values": ["High", "Medium", "Low"], "derived": true},
    {"name": "LastModifiedDate", "type": "date", "nullable": false},
    {"name": "LoadDateTime", "type": "timestamp", "nullable": false},
    {"name": "SourceFileDate", "type": "date", "nullable": false},
    {"name": "DataQualityFlag", "type": "boolean", "nullable": false},
    {"name": "ProcessingStatus", "type": "string", "nullable": false},
    {"name": "IntegrationLoadDateTime", "type": "timestamp", "nullable": false},
    {"name": "LineageVersion", "type": "integer", "nullable": false, "default": 1},
    {"name": "DataQualityScore", "type": "decimal(5,2)", "nullable": false, "range": [0, 100]}
  ],
  "write_mode": "overwrite",
  "quality_gates": {
    "null_tolerance_percent": 5,
    "min_quality_score": 80,
    "min_completeness_percent": 95
  }
}
```

**Warehouse Schema** (wh_cl):
```json
{
  "database": "consumption",
  "table_name": "DimCustomer",
  "type": "dimension_scd_type2",
  "storage_format": "warehouse_relational",
  "location": "wh_cl.consumption.DimCustomer",
  "primary_key": ["CustomerSK"],
  "business_key": ["CustomerID"],
  "partition_by": ["EffectiveDate"],
  "columns": [
    {"name": "CustomerSK", "type": "bigint", "constraint": "PRIMARY KEY", "identity": true, "seed": 1, "increment": 1, "description": "Surrogate key (auto-generated)"},
    {"name": "CustomerID", "type": "varchar(50)", "nullable": false, "description": "Business key from source"},
    {"name": "FirstName", "type": "varchar(100)", "nullable": true},
    {"name": "LastName", "type": "varchar(100)", "nullable": true},
    {"name": "Email", "type": "varchar(255)", "nullable": false},
    {"name": "Phone", "type": "varchar(20)", "nullable": true},
    {"name": "Country", "type": "varchar(50)", "nullable": true},
    {"name": "Status", "type": "varchar(20)", "nullable": false},
    {"name": "CustomerSegment", "type": "varchar(10)", "nullable": true},
    {"name": "RegistrationDate", "type": "date", "nullable": true},
    {"name": "EffectiveDate", "type": "date", "nullable": false, "scd_type": 2, "description": "Start date of SCD record"},
    {"name": "EndDate", "type": "date", "nullable": true, "scd_type": 2, "description": "End date (null = current)"},
    {"name": "IsCurrent", "type": "tinyint", "nullable": false, "scd_type": 2, "values": [0, 1], "description": "1=current row, 0=historical"},
    {"name": "LoadDateTime", "type": "datetime2", "nullable": false, "description": "Audit: when row loaded"},
    {"name": "SourceFileDate", "type": "date", "nullable": false, "description": "Audit: source batch date"}
  ],
  "merge_logic": "MERGE ON CustomerID; if Status/Segment changed: close previous row (EndDate, IsCurrent=0), insert new row (EffectiveDate, IsCurrent=1)"
}
```

### 3. Interface Contracts (Public APIs)

**Notebook Parameters** (inputs to each notebook):

```python
# LandingToRaw.py inputs
LANDING_PATH = "/Files/data/"  # OneLake path for CSV files
RAW_TABLE_NAME = "tbl_customer_raw"  # Target delta table
RAW_LAKEHOUSE = "lh_rl"  # Target lakehouse
ENVIRONMENT = "dev"  # Environment (dev/test/prod)

# RawToIntegration.py inputs
RAW_TABLE_NAME = "tbl_customer_raw"
RAW_LAKEHOUSE = "lh_rl"
INTEGRATION_TABLE_NAME = "tbl_customer_integration"
INTEGRATION_LAKEHOUSE = "lh_il"
ENVIRONMENT = "dev"

# IntegrationToWarehouse.py inputs
INTEGRATION_TABLE_NAME = "tbl_customer_integration"
INTEGRATION_LAKEHOUSE = "lh_il"
WAREHOUSE_TABLE = "consumption.DimCustomer"
WAREHOUSE_NAME = "wh_cl"
ENVIRONMENT = "dev"
```

**Error Codes** (for recovery procedures):

```
SUCCESS: 0 (pipeline completed)
CRITICAL_FILE_NOT_FOUND: 1001 (Landing CSV not found)
CRITICAL_SCHEMA_MISMATCH: 1002 (Raw schema validation failed)
CRITICAL_PIPELINE_TIMEOUT: 1003 (Transform exceeded SLA)
CRITICAL_IDENTITY_VIOLATION: 1004 (Warehouse surrogate key duplicate)
WARNING_DATA_QUALITY_THRESHOLD: 2001 (Quality score <80%)
WARNING_ROW_COUNT_VARIANCE: 2002 (Row count variance >20%)
INFO_DUPLICATE_DETECTED: 3001 (Duplicate email found)
```

### 4. Quickstart Guide

**Getting Started** (5 steps):

1. **Setup Notebooks in Microsoft Fabric Workspace** (`ws_rritec_dev`):
   - Import `LandingToRaw.py`, `RawToIntegration.py`, `IntegrationToWarehouse.py`
   - Import `AuditUtils.py`, `BackupUtils.py`, `AuditLogger.py`
   - Attach to default Microsoft Fabric cluster

2. **Create Lakehouses & Warehouse**:
   ```
   ws_rritec_dev:
     - lh_rl (Raw Layer)
     - lh_il (Integration Layer)
     - lh_audit (Audit & Logging)
     - wh_cl (Warehouse)
   ```

3. **Create Sample Data**:
   - Download `test_data.csv` (7-day sample customer file)
   - Place in OneLake `/Files/data/` folder
   - Rename to `customer_2026-04-13.csv`

4. **Run Pipeline (Step-by-Step)**:
   ```
   Step 1: Execute LandingToRaw.py
   Step 2: Validate tbl_customer_raw created
   Step 3: Execute RawToIntegration.py
   Step 4: Validate tbl_customer_integration (deduplicated)
   Step 5: Execute IntegrationToWarehouse.py
   Step 6: Validate DimCustomer loaded (with surrogate keys)
   Step 7: Query audit tables in lh_audit
   ```

5. **Troubleshooting**:
   - Check `pipeline_execution_log` for errors
   - Review `data_quality_metrics` for quality issues
   - Inspect `lineage_tracking` for dependency flow
   - Use notebooks error codes (see section 3 above)

---

## Phase 1 Artifacts Summary

**Outputs Created**:
- ✅ `data-model.md` (entity definitions, relationships, versioning)
- ✅ Contracts (4 JSON schemas: Landing, Raw, Integration, Warehouse)
- ✅ `quickstart.md` (5-step setup & troubleshooting)
- ✅ **Agent context updated** (Microsoft Fabric, PySpark, delta lake technologies)

**Next Phase**: Phase 2 will generate task list from this plan.

---

## Glossary & References

| Term | Definition |
|------|-----------|
| **Landing** | Raw CSV storage layer (OneLake `/Files/data/`) |
| **Raw | Immutable all-data persistence with audit columns (lh_rl) |
| **Integration** | Business transformations & enrichment (lh_il) |
| **Warehouse** | BI-ready consumption layer with surrogates & SCD (wh_cl) |
| **SCD Type-2** | Slowly Changing Dimension: track history via EffectiveDate/EndDate/IsCurrent |
| **Surrogate Key** | Auto-generated artificial key (e.g., CustomerSK identity column) |
| **Audit Columns** | LoadDateTime, SourceFileDate, DataQualityFlag, ProcessingStatus |
| **Deduplication** | Remove duplicates; retain latest by timestamp |
| **Microsoft Fabric** | Data analytics platform with OneLake, Notebooks, Warehouse, Pipelines |
| **PySpark** | Python API for Apache Spark (distributed processing) |
| **Delta Lake** | ACID-compliant table format for data lakes |

---

**Plan Complete** ✅

**Version**: 1.0 | **Branch**: `001-customer-mdm-pipeline` | **Status**: Ready for Phase 2 Task Generation
