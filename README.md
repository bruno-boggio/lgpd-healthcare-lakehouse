# LGPD Healthcare Lakehouse - Project Summary

## Overview

**Project Name:** LGPD Healthcare Lakehouse  
**Domain:** Healthcare Data Engineering  
**Cloud Platform:** Microsoft Azure  
**Data Platform:** Azure Databricks  
**Orchestration:** Azure Data Factory  
**Architecture:** Medallion (Bronze/Silver/Gold)  
**Compliance:** LGPD (Data Privacy Law)

---

## Project Objectives

1. Build a production-grade data lakehouse for healthcare analytics
2. Implement LGPD compliance with PII data segregation
3. Create automated ETL pipelines with monitoring and alerts
4. Enable secure data access with role-based controls
5. Demonstrate enterprise-level data engineering practices

---

##  Architecture

### Medallion Architecture (3 Layers)

```
Landing (CSV files)
    ↓
Bronze (Raw Delta Lake)
    ↓
Silver (Cleaned & Validated)
    ↓
Gold (Business Logic & Analytics)
```

### Technology Stack

- **Storage:** Azure Data Lake Storage Gen2 (ADLS)
- **Compute:** Azure Databricks (Spark)
- **Orchestration:** Azure Data Factory (ADF)
- **Format:** Delta Lake
- **Catalog:** Hive Metastore
- **Alerting:** Azure Logic Apps
- **Language:** Python, PySpark, SQL

---

## 📊 Data Model

### Star Schema Design

**Dimensions (6):**
- `dim_data` - Date dimension
- `dim_clinica` - Clinic/healthcare facility
- `dim_medico` - Doctor (pseudonymized with tokens)
- `dim_diagnostico` - Diagnosis (ICD codes)
- `dim_exame` - Medical exams
- `dim_paciente` - Patient (pseudonymized with tokens)

**Fact Table (1):**
- `fato_consultas` - Medical consultations (grain: one row per consultation)

**PII Tables (2) - Separate secured storage:**
- `paciente_identidade` - Patient PII (CPF, name, email, phone)
- `medico_identidade` - Doctor PII (CPF, name, CRM)

### LGPD Compliance Strategy

**Pseudonymization:**
- Dimensions contain only tokens (no direct identifiers)
- PII stored separately in `/pii/` folder with restricted access
- Tokenization maintained for joins between fact/dimensions

**Physical Separation:**
- Regular data: `/bronze/`, `/silver/`, `/gold/`
- Sensitive data: `/bronze/pii/`, `/silver/pii/`
- Access control via Azure RBAC and ACLs

---

##  ETL Pipelines

### Landing → Bronze (Ingestion)

**3 Notebooks:**
1. `01_ingest_dimensions` - Processes 6 dimension tables
2. `02_ingest_pii` - Processes 2 PII tables (separate path)
3. `03_ingest_facts` - Processes fact table

**Features:**
- Dynamic file detection (only processes existing files)
- Delta Lake format with ACID transactions
- Schema evolution enabled (`mergeSchema: true`)
- Audit columns: `ingestion_timestamp`, `ingestion_date`
- File management: moves processed files to `processed/` or `failed/` folders
- Error handling with detailed logging

**ADF Pipelines (3):**
- `01_Ingest_Dimensions_To_Bronze`
- `02_Ingest_Identities_To_Bronze`
- `03_Ingest_Fact_To_Bronze`

**Pipeline Components:**
- Set Variable (generate execution_id)
- Get Metadata (check for files in landing)
- If Condition (process only if files exist)
- Databricks Notebook activities
- Logging notebooks (start/success/failure)
- Web activity (email alerts on failure)

### Bronze → Silver (Transformation)

**3 Notebooks:**
1. `01_transform_dimensions` - Cleans and validates 6 dimensions
2. `02_transform_pii` - Validates PII data
3. `03_transform_facts` - Validates fact table with FK checks

**Transformations Applied:**

**Data Cleaning:**
- Trim whitespace
- Uppercase/lowercase standardization
- Remove special characters
- Clean phone numbers (digits only)
- Clean CPF (11 digits validation)

**Data Validation:**
- Type casting (int, decimal, date)
- Null value handling
- Range validation (year between 1900-2026, etc.)
- FK validation (all foreign keys must exist)
- Business rule validation (plano_cobriu: 0/1, ativo: 0/1)

**Data Quality:**
- Duplicate removal (keep most recent by ingestion_timestamp)
- Invalid record rejection with logging
- Metrics: input count, output count, rejection rate

**Storage Pattern:**
- MERGE (upsert) instead of append
- Prevents duplicates on reprocessing
- Key-based updates

**ADF Pipelines (3):**
- `pl_transform_dimensions_to_silver`
- `pl_transform_pii_to_silver`
- `pl_transform_facts_to_silver`

---

## Logging & Monitoring

### Audit Table

**Location:** `healthcare_bronze.pipeline_execution_log`

**Schema:**
```sql
execution_id STRING          -- Unique GUID per execution
pipeline_name STRING          -- ADF pipeline name
start_time TIMESTAMP         -- Execution start
end_time TIMESTAMP           -- Execution end
status STRING                -- RUNNING/SUCCESS/FAILED
records_processed INT        -- Count of records
error_message STRING         -- Error details if failed
run_date DATE               -- Partition key
executed_by STRING          -- Trigger type (Manual/Schedule)
```

### Logging Notebooks (3)

1. **`log_pipeline_start`**
   - Inserts record with status='RUNNING'
   - Captures execution_id, pipeline_name, executed_by

2. **`log_pipeline_success`**
   - Updates record with status='SUCCESS'
   - Records end_time and records_processed

3. **`log_pipeline_failure`**
   - Updates record with status='FAILED'
   - Captures error_message for troubleshooting

### Email Alerts

**Failure Alerts:**
- Triggered on pipeline failure
- Azure Logic App sends formatted email
- Contains: pipeline name, error message, timestamp, Databricks run URL

**Daily Report:**
- Scheduled pipeline: `pl_send_daily_report`
- Aggregates execution metrics
- Email includes:
  - Total executions
  - Success/failure breakdown
  - Per-pipeline statistics
  - Records processed

---

##  Security & Compliance

### LGPD Implementation

**1. Data Minimization:**
- Only necessary PII collected
- Pseudonymization with tokens in analytical tables

**2. Physical Separation:**
- PII stored in separate folders (`/pii/`)
- Restricted access via RBAC

**3. Access Control (Attempted):**
- Service Principal created for restricted access
- ACLs configured on `/pii/` folders
- Proof of concept for production implementation

**4. Data Retention:**
- Processed files archived by date
- Failed files preserved for investigation

## 📦 Project Structure

```
lgpd-healthcare-lakehouse/
├── notebooks/
│   ├── setup/
│   │   └── create_databases_and_tables.py
│   ├── bronze/
│   │   ├── 01_ingest_dimensions.py
│   │   ├── 02_ingest_identities.py
│   │   └── 03_ingest_facts.py
│   ├── silver/
│   │   ├── 01_transform_dimensions.py
│   │   ├── 02_transform_identities.py
│   │   ├── 03_transform_facts.py
│   │   └── 04_mask_pii.py                    
│   ├── gold/
│   │   ├── 01_aggregate_metrics.py           
│   │   ├── 02_aggregate_reports.py           
│   │   └── 03_visualize_dashboard.py         
│   └── logs/
│       ├── log_pipeline_start.py
│       ├── log_pipeline_success.py
│       ├── log_pipeline_failure.py
│       └── generate_daily_report.py
├── adf/
│   └── pipelines/
│       ├── Master_Pipeline_Bronze.json
│       ├── Master_Pipeline_Silver.json
│       ├── Master_Pipeline_Gold.json         
│       └── pl_send_daily_report.json
└── README.md
```

### Storage Structure

```
Azure Data Lake Storage Gen2
├── landing/                    # CSV source files
│   ├── processed/             # Successfully processed (by date)
│   └── failed/                # Failed processing (by date)
├── bronze/                    # Raw Delta Lake
│   ├── dim_data/
│   ├── dim_clinica/
│   ├── dim_medico/
│   ├── dim_diagnostico/
│   ├── dim_exame/
│   ├── dim_paciente/
│   ├── facts/
│   │   └── fato_consultas/
│   ├── pii/                   # Restricted access
│   │   ├── paciente_identidade/
│   │   └── medico_identidade/
│   └── control/
│       └── pipeline_execution_log/
├── silver/                    # Cleaned Delta Lake
│   ├── dim_data/
│   ├── dim_clinica/
│   ├── dim_medico/
│   ├── dim_diagnostico/
│   ├── dim_exame/
│   ├── dim_paciente/
│   ├── facts/
│   │   └── fato_consultas/
│   └── pii/                   # With masked columns
│       ├── paciente_identidade/
│       └── medico_identidade/
└── gold/                      # Business logic      
    ├── agg_consultas_por_periodo/
    ├── agg_consultas_por_medico/
    ├── agg_consultas_por_clinica/
    ├── agg_consultas_por_diagnostico/
    ├── agg_performance_exames/
    ├── agg_resumo_especialidade/
    ├── agg_perfil_pacientes/
    ├── rpt_top_medicos_receita/
    ├── rpt_pacientes_alto_gasto/
    ├── rpt_medicos_por_clinica/
    ├── rpt_historico_pacientes/
    ├── rpt_medicos_pacientes_cidade/
    └── rpt_contato_pacientes_vip/
```

---

### Template Method Pattern
Fixed processing skeleton with variable transformations:
```python
def process_to_silver():
    1. Read from Bronze (fixed)
    2. Transform (varies - Strategy)
    3. Deduplicate (fixed)
    4. Add metadata (fixed)
    5. MERGE to Silver (fixed)
```

## 🔧 Technical Highlights

### Delta Lake Features
- **ACID Transactions:** Guaranteed consistency
- **Time Travel:** Version history for rollback
- **Schema Evolution:** Add columns without breaking
- **MERGE Operations:** Upsert for idempotency

### PySpark Best Practices
- Type hints for functions
- Reusable utility functions
- Error handling with try/except
- Lazy evaluation optimization
- Avoid unnecessary counts (performance)

---

## 📈 Operational Metrics

**Captured Automatically:**
- Pipeline execution count
- Success/failure rates
- Records processed per run
- Processing duration
- Error messages and stack traces

**Available for Analysis:**
- Daily execution trends
- Pipeline reliability metrics
- Data volume growth
- Error patterns

---

## 🚀 Deployment

**Current State:** Development/POC
- Manual trigger via ADF
- Single Databricks cluster
- Hive Metastore
  
---

## 📚 Skills Demonstrated

### Data Engineering
✅ Medallion architecture implementation  
✅ ETL pipeline design and development  
✅ Data modeling (star schema)  
✅ Data quality and validation  
✅ Incremental processing patterns  

### Azure Cloud
✅ Azure Data Lake Storage Gen2  
✅ Azure Databricks (Spark)  
✅ Azure Data Factory orchestration  
✅ Azure Logic Apps integration  
✅ Service Principal authentication  

### Programming
✅ Python/PySpark  
✅ SQL (queries and DDL)  
✅ Design patterns (Strategy, Template Method)  
✅ Error handling and logging  
✅ Code organization and modularity  

### Compliance & Governance
✅ LGPD data protection principles  
✅ PII identification and segregation  
✅ Access control design (RBAC, ACLs)  
✅ Data retention policies  

### DevOps/Operational
✅ Pipeline monitoring and alerting  
✅ Email notifications with Logic Apps  
✅ Execution logging and metrics  
✅ Daily operational reports  
✅ Error handling and recovery  

---

## 💼 Business Value

**For Healthcare Organizations:**
- Centralized patient data for analytics
- LGPD-compliant data handling
- Automated data pipelines (reduce manual work)
- Audit trail for compliance reporting
- Scalable architecture for growth
- 100% data lineage tracking
- Zero data loss with ACID transactions

