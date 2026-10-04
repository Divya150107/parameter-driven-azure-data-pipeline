# Parameter-Driven Azure Bronze Data Ingestion Pipeline

A reusable and parameter-driven Azure Data Factory pipeline designed to incrementally ingest Spotify data from Azure SQL Database into an Azure Data Lake Storage Gen2 Bronze layer in Parquet format.

The project focuses on building a reusable cloud data ingestion framework that can process multiple source tables using the same pipeline logic.

---

## Project Overview

This project implements a Bronze-layer data ingestion pipeline using Azure Data Factory.

The source data consists of Spotify-related tables stored in Azure SQL Database. Instead of creating a separate pipeline for every table, the solution uses a parameterized configuration to dynamically determine:

- Source schema
- Source table
- CDC / watermark column
- Optional historical starting date

The pipeline then performs incremental ingestion and stores the extracted data in the Bronze layer of Azure Data Lake Storage Gen2.

The project intentionally focuses on the **Bronze ingestion layer** and does not include Silver or Gold transformations.

---

## Architecture

```text
                    Azure SQL Database
                           |
                           | Incremental SQL Query
                           v
                  Azure Data Factory
                           |
                  Parameterized Input
                           |
                           v
                       ForEach
                           |
             +-------------+-------------+
             |                           |
       CDC / Watermark              Table Configuration
             |                           |
             +-------------+-------------+
                           |
                           v
                    Copy Activity
                           |
                           v
              Azure Data Lake Storage Gen2
                           |
                        Bronze
                           |
        +------------------+------------------+
        |                  |                  |
     DimUser           DimTrack          DimArtist
        |                  |                  |
     DimDate          FactStream              |
        |                  |                  |
        +------------------+------------------+
```

---

## Azure Services Used

### Azure Data Factory

Used for:

- Pipeline orchestration
- Parameterization
- Dynamic SQL generation
- Incremental data extraction
- Data movement
- Control flow
- Failure handling

### Azure Data Lake Storage Gen2

Used as the Bronze storage layer.

Extracted source data is stored as Parquet files in table-specific folders.

### Azure SQL Database

Acts as the source database containing the Spotify-related tables.

### Azure Logic Apps

Used for failure notification.

When the ingestion pipeline fails, Azure Data Factory sends a POST request to a Logic App, which can trigger an email notification.

---

## Key Features

### 1. Parameter-Driven Table Processing

The pipeline accepts an array of table configurations.

Each configuration contains:

```text
schema
table
cdc_col
from_date
```

Example:

```json
{
    "schema": "dbo",
    "table": "DimUser",
    "cdc_col": "updated_at",
    "from_date": ""
}
```

The same pipeline can therefore process multiple tables without creating separate pipelines for each table.

---

### 2. Incremental Loading

The pipeline uses a watermark-based incremental loading approach.

For each table:

```text
Previous CDC Value
        |
        v
Determine Load Start Point
        |
        v
Query New Records
        |
        v
Write to Bronze
        |
        v
Calculate Latest CDC
        |
        v
Update CDC State
```

The source query is dynamically constructed using the configured CDC column.

Conceptually:

```sql
SELECT *
FROM <schema>.<table>
WHERE <cdc_column> > <previous_cdc_value>;
```

This allows normal pipeline executions to process only records newer than the previously stored watermark.

---

### 3. CDC / Watermark State Tracking

The pipeline maintains the latest processed CDC value for each table.

The state is stored in the Bronze storage area as:

```text
bronze/
├── DimUser_cdc/
│   └── cdc.json
├── DimTrack_cdc/
│   └── cdc.json
├── DimArtist_cdc/
│   └── cdc.json
├── DimDate_cdc/
│   └── cdc.json
└── FactStream_cdc/
    └── cdc.json
```

During a pipeline run:

1. The previous CDC value is read.
2. New records are extracted.
3. The maximum CDC value is calculated.
4. The CDC file is updated after successful data ingestion.

This allows the next execution to continue from the latest processed point.

---

### 4. Historical / Backdated Loading

The pipeline supports controlled historical loading through the `from_date` parameter.

If `from_date` is provided, the pipeline uses it as the starting point instead of the previously stored CDC value.

For example:

```json
{
    "schema": "dbo",
    "table": "FactStream",
    "cdc_col": "stream_timestamp",
    "from_date": "2025-01-01"
}
```

This allows historical data to be reprocessed without changing the pipeline logic.

Typical use cases include:

- Recovering missed historical data
- Initial historical ingestion
- Reprocessing a specific historical period

---

### 5. Dynamic Bronze Storage

The destination path is generated dynamically based on the table being processed.

The Bronze layer follows a table-oriented structure:

```text
bronze/
│
├── DimUser/
│   └── DimUser_<timestamp>.parquet
│
├── DimTrack/
│   └── DimTrack_<timestamp>.parquet
│
├── DimArtist/
│   └── DimArtist_<timestamp>.parquet
│
├── DimDate/
│   └── DimDate_<timestamp>.parquet
│
└── FactStream/
    └── FactStream_<timestamp>.parquet
```

This avoids hardcoding separate destination paths for every table.

---

### 6. Parquet Storage

The Bronze data is stored in **Parquet** format.

Parquet was selected because it is:

- Columnar
- Efficient for analytical workloads
- More storage-efficient than plain text formats
- Well supported by Azure data services
- Suitable for downstream data processing

The Bronze layer therefore preserves the ingested source data while storing it in an analytics-friendly file format.

---

### 7. Empty Increment Handling

Not every pipeline execution will necessarily have new records.

The pipeline checks the number of records read by the Copy Activity.

```text
Copy Activity
      |
      v
Data Read > 0 ?
    /       \
  Yes        No
   |          |
   v          v
Update CDC   Delete
             Empty File
```

If no records are available:

- The generated empty Parquet file is deleted.
- The previous CDC value remains unchanged.

This prevents unnecessary empty files from accumulating in the Bronze layer.

---

### 8. Failure Notification

The pipeline contains a failure-handling path using an Azure Data Factory Web Activity.

```text
ADF Pipeline
     |
     v
ForEach Processing
     |
     X
   Failure
     |
     v
Web Activity
     |
     v
Azure Logic App
     |
     v
Email Notification
```

The notification payload contains information such as:

```text
Pipeline Name
Pipeline Run ID
```

This provides a simple automated mechanism for being notified when ingestion fails.

---

## Tables Configured

The current pipeline configuration contains five Spotify-related tables:

| Table | CDC / Watermark Column |
|---|---|
| DimUser | updated_at |
| DimTrack | updated_at |
| DimDate | date |
| DimArtist | updated_at |
| FactStream | stream_timestamp |

The CDC column is configurable for each table.

---

## Pipeline Execution Flow

The overall process is:

```text
1. Receive table configuration
              |
              v
2. Iterate through tables using ForEach
              |
              v
3. Read previous CDC value
              |
              v
4. Determine starting point
              |
        +-----+------+
        |            |
   from_date      previous CDC
   provided?        value
        |            |
        +-----+------+
              |
              v
5. Generate dynamic SQL query
              |
              v
6. Extract records from Azure SQL
              |
              v
7. Write records as Parquet
              |
              v
8. Check number of records loaded
              |
        +-----+------+
        |            |
      > 0            0
        |            |
        v            v
9. Calculate       Delete empty
   max CDC          output file
        |
        v
10. Update CDC state
```

---

## Example Dynamic SQL

The pipeline dynamically constructs a query based on the current table configuration.

For a normal incremental load:

```sql
SELECT *
FROM dbo.DimUser
WHERE updated_at > '<previous_cdc_value>';
```

For a historical load:

```sql
SELECT *
FROM dbo.DimUser
WHERE updated_at > '<from_date>';
```

The actual schema, table, CDC column, and starting value are supplied dynamically by the pipeline.

---

## Pipeline Components

The project contains two main ingestion pipelines.

### `incremental_loop`

This pipeline acts as the main orchestration layer.

It:

- Receives the table configuration array
- Iterates through the configured tables
- Executes the incremental ingestion logic
- Handles empty loads
- Updates CDC state
- Triggers failure notification

### `incremental_ingestion`

This pipeline contains the reusable table-level incremental ingestion logic.

It accepts parameters such as:

```text
schema
table
cdc_col
from_date
```

The pipeline uses these parameters to dynamically construct the source query and destination path.

---

## Dataset Design

The project uses parameterized datasets so that the same dataset definitions can be reused for multiple tables and files.

### Azure SQL Dataset

Used to connect the ingestion pipeline to the Azure SQL source.

### Dynamic JSON Dataset

Used for reading and writing CDC state files.

Conceptually:

```text
bronze/
└── <table>_cdc/
    └── cdc.json
```

### Dynamic Parquet Dataset

Used for dynamically writing Bronze output files.

Conceptually:

```text
bronze/
└── <table>/
    └── <table>_<timestamp>.parquet
```

---

## Repository Structure

```text
parameter-driven-azure-data-pipeline/
│
├── README.md
│
├── dataset/
│   ├── azure_sql.json
│   ├── json_dynamic.json
│   └── parquet_dynamic.json
│
├── factory/
│   └── divya-dfazureproject.json
│
├── linkedService/
│   ├── azure_sql.json
│   └── datalake.json
│
├── pipeline/
│   ├── incremental_ingestion.json
│   └── incremental_loop.json
│
└── publish_config.json
```

---

## Design Principles

The pipeline was designed around the following principles.

### Reusability

One pipeline can process multiple tables instead of creating separate ingestion pipelines for every table.

### Parameterization

Table-specific information is supplied dynamically rather than hardcoded throughout the pipeline.

### Incremental Processing

Normal executions extract only records newer than the stored watermark.

### Backfill Support

Historical data can be explicitly reprocessed using a configurable starting date.

### Dynamic Storage

Table names and execution timestamps are used to generate Bronze output paths and filenames.

### State Tracking

The latest CDC value is persisted between pipeline executions.

### Failure Handling

Pipeline failures can trigger automated notifications.

### Bronze Preservation

The Bronze layer focuses on ingestion and storage rather than applying business transformations.

---

## Why Bronze?

The Bronze layer is responsible for capturing data from the source system with minimal transformation.

For this project, the Bronze layer provides:

- Raw source data preservation
- Historical ingestion files
- Incremental ingestion storage
- A foundation for future downstream processing

The project intentionally stops at the Bronze layer so that the ingestion architecture can be understood and demonstrated independently.

---

## Scope

This project intentionally focuses on the **Bronze data ingestion layer**.

### Included

- Azure Data Factory orchestration
- Parameter-driven ingestion
- Incremental loading
- CDC / watermark tracking
- Historical backfill support
- Dynamic datasets
- Parquet storage
- Empty-load handling
- Failure notification
- Azure Logic Apps integration

### Not Included

- Silver-layer transformations
- Gold-layer transformations
- Power BI dashboards
- Machine learning
- Advanced data quality frameworks

The objective is to demonstrate the design and implementation of a reusable cloud-based data ingestion pipeline.

---

## Production Improvements

For a production environment, the following improvements could be introduced:

### Secret Management

Move connection secrets and credentials to **Azure Key Vault** instead of storing them directly in linked-service configurations.

### External Metadata Store

Move table configuration from pipeline parameters to an external metadata/control table.

For example:

```text
table_name
schema_name
cdc_column
source_system
active_flag
backfill_start_date
```

This would allow ingestion behavior to be changed without modifying the pipeline definition.

### Centralized Logging

Maintain a control/log table containing:

```text
pipeline_name
pipeline_run_id
table_name
start_time
end_time
records_read
status
error_message
```

### Monitoring

Integrate Azure Data Factory with Azure Monitor and Log Analytics for centralized monitoring and alerting.

### Data Quality

Add validation checks for:

- Null values
- Duplicate records
- Invalid timestamps
- Unexpected schema changes
- Record-count anomalies

### Parallel Processing

The current ForEach processing is configured sequentially.

For independent tables, controlled parallel execution could be introduced to improve performance while avoiding excessive load on the source database.

### CI/CD

Implement CI/CD using GitHub Actions or Azure DevOps to deploy ADF changes across development, testing, and production environments.

---

## Security

The public repository intentionally does not contain:

- Database passwords
- Access keys
- API keys
- Logic App callback signatures
- Other authentication secrets

Connection-specific credentials are excluded from the public portfolio repository.

---

## Technologies

```text
Azure Data Factory
Azure Data Lake Storage Gen2
Azure SQL Database
Azure Logic Apps
SQL
Parquet
JSON
Git
GitHub
```

---

## Project Objective

The objective of this project is to demonstrate how a reusable Azure Data Factory pipeline can ingest multiple source tables into an ADLS Gen2 Bronze layer using:

- Parameterization
- Incremental processing
- CDC / watermark state tracking
- Historical backfills
- Dynamic datasets
- Dynamic file organization
- Parquet storage
- Empty-load handling
- Automated failure notification

The project emphasizes core data engineering concepts such as **cloud orchestration, incremental ingestion, parameterization, state management, data lake storage, and failure handling**.
