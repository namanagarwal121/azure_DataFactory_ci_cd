# Azure Data Factory CI/CD

A Git-integrated Azure Data Factory (`DFProdfnaman`) containing several ingestion patterns built on Azure Data Lake Storage Gen2 and Azure SQL Database: event-style file routing, watermark-based incremental loading from SQL, paginated REST API ingestion, and a Mapping Data Flow that upserts into a Delta table. Changes are developed on a feature branch and merged into `main` through pull requests, with `adf_publish` configured as the publish branch.

## Architecture / Flow

```text
ADLS Gen2 source/files/*.csv ──> Router (validate trigger file, switch by file name) ──> sink/{customers|drivers|trips}/
Azure SQL (source.Orders)   ──> Ingestion (watermark in source/monitor/cdc.json)    ──> sink/Orders/ (Parquet)
PokeAPI (REST)              ──> RestApi (offset pagination)                         ──> sink/API/pokemon.json
ADLS Gen2 source/deltasource/locations.csv ──> delta lake (deltaflow data flow)      ──> sink/deltasink (Delta, upsert)
```

## Tech stack

- Azure Data Factory (pipelines, Mapping Data Flows), Git integration
- Azure Data Lake Storage Gen2 (`source` and `sink` containers)
- Azure SQL Database
- REST source (public PokeAPI, anonymous)
- File formats: CSV, JSON, Parquet, Delta

## Repository structure

```text
.
├── factory/DFProdfnaman.json     # Factory definition (Central India, system-assigned identity)
├── pipeline/                     # Router, Ingestion, Scheduled, RestApi, delta lake
├── dataflow/deltaflow.json       # Mapping Data Flow writing to Delta with upsert
├── dataset/                      # CSV, JSON, Parquet, Azure SQL and REST datasets
├── linkedService/                # ls_storage (ADLS Gen2), ls_db_connection (Azure SQL), ls_Rest (REST)
└── publish_config.json           # Publish branch: adf_publish
```

## Pipelines

### Router
1. **Validate** waits (polling every 10 s, up to 12 h) for the trigger file `source/trigger/locations.csv`.
2. **Get Metadata** lists the files in `source/files`.
3. **ForEachFile** runs a **Switch** on the file name prefix (text before the first `.`):
   - `customers` -> copied to `sink/customers/`
   - `drivers` -> copied to `sink/drivers/`
   - `trips` -> copied to `sink/trips/`
4. **Delete** removes the trigger file once all files are processed.

All copies use the parameterised dataset `ds_csv_dynamic` (`p_container`, `p_folder`, `p_file`).

### Ingestion (incremental load from Azure SQL)
Parameters: `schema` (default `source`), `table` (default `Orders`), `backdate` (optional override).

1. **timestamp** (Lookup) reads the last watermark from `source/monitor/cdc.json`.
2. **last_run** stores it in the `time_stamp` variable.
3. **Totalresults** (Script) queries rows where `last_updated` is greater than the watermark, or than `backdate` if supplied.
4. **If Condition1**: when rows were returned,
   - **Copy data** writes the new rows to `sink/Orders/` as Snappy Parquet,
   - **maxcdc** (Script) gets `max(last_updated)` from the table,
   - **change_cdc** copies `source/monitor/empty.json` with an added `cdc_timestamp` column to overwrite `cdc.json`, advancing the watermark.

### Scheduled
Wraps **Ingestion** in an Execute Pipeline activity (wait on completion). No trigger definition is stored in this repository.

### RestApi
1. **Endpoint** (Web activity) calls `GET https://pokeapi.co/api/v2/pokemon` to read the total `count`.
2. **CopyAllData** copies from the REST dataset using an `offset` query-parameter pagination rule (`RANGE:0:<count>:20`) and writes JSON to `sink/API/pokemon.json`.

### delta lake
Runs the **deltaflow** Mapping Data Flow (General compute, 8 cores) with source `source/deltasource/locations.csv`:

- `source` reads location records (source row limit set to 100), schema drift allowed
- `upperState` upper-cases `state`
- `selectCols` keeps `location_id`, `city`, `state`, `country`, `last_updated_timestamp`
- `alterRow` marks every row for upsert
- `sinkDelta` writes an inline Delta sink to `sink/deltasink`, upserting on key `location_id`

## CI/CD

- ADF Studio is connected to this repository; factory resources are committed as JSON on the working branch.
- Feature work was done on `naman/feat` and merged into `main` via pull request.
- `publish_config.json` sets `adf_publish` as the publish branch, where ADF generates ARM templates on publish. That branch is not present in this repository, and no ARM templates or Azure DevOps / GitHub Actions pipeline definitions are included.

## Prerequisites

- An ADLS Gen2 account with `source` and `sink` containers and the folders referenced above (`files`, `trigger`, `monitor`, `deltasource`)
- A seed `source/monitor/cdc.json` containing a `cdc_timestamp` value and an `source/monitor/empty.json` file
- An Azure SQL Database with a table containing a `last_updated` column (default `source.Orders`)
- Credentials for the storage and SQL linked services configured in ADF (they are not stored in this repository)
