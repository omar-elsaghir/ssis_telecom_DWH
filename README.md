# 📡 Telecom Data Warehouse — SSIS ETL Pipeline

> **End-to-end ETL pipeline** that ingests daily telecom batch CSV files, cleanses the data, validates subscribers against a dimension table, and loads clean records into a SQL Server fact table — while routing bad rows to an audit table instead of crashing the pipeline.

---

## 📋 Table of Contents

1. [Project Overview](#-project-overview)
2. [Tech Stack](#️-tech-stack)
3. [Architecture Diagram](#-architecture-diagram)
4. [Prerequisites](#-prerequisites)
5. [Step 1 — Database Setup](#step-1--database-setup)
6. [Step 2 — Understand the Source Data](#step-2--understand-the-source-data)
7. [Step 3 — Create the SSIS Project](#step-3--create-the-ssis-project)
8. [Step 4 — Configure the Foreach Loop Container](#step-4--configure-the-foreach-loop-container)
9. [Step 5 — Configure the Flat File Connection Manager](#step-5--configure-the-flat-file-connection-manager)
10. [Step 6 — Build the Data Flow Task](#step-6--build-the-data-flow-task)
11. [Step 7 — Configure the Lookup Transformation](#step-7--configure-the-lookup-transformation)
12. [Step 8 — Configure the Derived Column Transformation](#step-8--configure-the-derived-column-transformation)
13. [Step 9 — Configure OLE DB Destinations](#step-9--configure-ole-db-destinations)
14. [Step 10 — Add the File System Task (Archive)](#step-10--add-the-file-system-task-archive)
15. [Step 11 — Run and Validate](#step-11--run-and-validate)
16. [Error Handling Strategy](#-error-handling-strategy)
17. [Repository Structure](#-repository-structure)
18. [Troubleshooting](#-troubleshooting)

---

## 📝 Project Overview

This pipeline automates the ingestion of **Call Detail Records (CDRs)** arriving as pipe-delimited CSV files. Each batch file follows the same schema:

```
id | imsi | imei | cell | lac | event_type | event_ts
```

| Column | Description |
|---|---|
| `id` | Source transaction identifier |
| `imsi` | International Mobile Subscriber Identity (up to 9 digits) |
| `imei` | International Mobile Equipment Identity (14 digits) |
| `cell` | Cell tower ID |
| `lac` | Location Area Code |
| `event_type` | Single-character event code (1–9) |
| `event_ts` | Event timestamp |

The pipeline handles real-world data quality issues including missing fields, type conversion failures, non-numeric IDs, and subscribers not found in the dimension table.

---

## 🛠️ Tech Stack

| Component | Technology |
|---|---|
| ETL Tool | SQL Server Integration Services (SSIS) |
| IDE | Visual Studio with SQL Server Data Tools (SSDT) |
| Database | Microsoft SQL Server (SQL Express) |
| Source Data | Pipe-delimited CSV flat files |
| Version Control | Git & GitHub |

---

## 🏗️ Architecture Diagram

```
 Source Files/
   └── *.csv (batch files)
          │
          ▼
 ┌────────────────────────────────────────────────┐
 │          Foreach Loop Container                 │
 │  (iterates each *.csv in Source Files folder)   │
 │                                                  │
 │  ┌──────────────────────────────────────────┐   │
 │  │           Data Flow Task                  │   │
 │  │                                           │   │
 │  │   [Flat File Source]                      │   │
 │  │         │  IgnoreFailure on bad rows      │   │
 │  │         ▼                                 │   │
 │  │   [Lookup Transformation]                 │   │
 │  │    Join imsi → dim_imsi_reference         │   │
 │  │         │                                 │   │
 │  │    Match Output    No-Match Output        │   │
 │  │         │               │                 │   │
 │  │         ▼               ▼                 │   │
 │  │  [Derived Column]  (discarded / audited)  │   │
 │  │   TAC, SNR, null-                         │   │
 │  │   safe subscriber                         │   │
 │  │         │                                 │   │
 │  │         ▼             Error Output        │   │
 │  │  [Fact_Transaction] ──────────────────►   │   │
 │  │   (OLE DB Dest.)    [error_destination]   │   │
 │  └──────────────────────────────────────────┘   │
 │                                                  │
 │  ┌──────────────────────────────────────────┐   │
 │  │   File System Task (Move to Processed/)   │   │
 │  └──────────────────────────────────────────┘   │
 └────────────────────────────────────────────────┘
```

> 📸 **Screenshot placeholder** — `images/image_a587a5.png` — *SSIS Data Flow canvas showing all components connected*

---

## ✅ Prerequisites

Before starting, ensure you have:

- **Visual Studio 2019/2022** with SQL Server Data Tools (SSDT) extension installed
- **SQL Server** (Express or higher) running locally
- **SQL Server Management Studio (SSMS)** for database setup
- Source CSV files placed in the `Source Files/` directory
- An empty `Processed Files/` directory at the same level

---

## Step 1 — Database Setup

Open SSMS and run the two SQL scripts in order.

### 1a. Create the database and tables

Open `SQL Queries/Create database.sql` and execute it. This creates:

- **`SSIS_Telecom_DB`** — the target data warehouse database
- **`fact_transaction`** — the main fact table that receives validated CDR rows
- **`error_destination_output`** — the audit table that catches rejected rows along with SSIS error codes

```sql
-- Key columns in fact_transaction
id              INT IDENTITY PRIMARY KEY   -- surrogate key (auto-generated)
transaction_id  INT                        -- source file's id column
imsi            VARCHAR(9)
subscriber_id   INT                        -- enriched from dim lookup
tac             VARCHAR(8)                 -- first 8 digits of IMEI
snr             VARCHAR(6)                 -- digits 9–14 of IMEI
imei            VARCHAR(14)
cell            INT
lac             INT
event_type      VARCHAR(1)
event_ts        DATETIME
```

> 📸 **Screenshot placeholder** — *SSMS showing the three tables created in Object Explorer*

### 1b. Create the IMSI dimension table and seed data

Open `SQL Queries/Create dim imsi.sql` and execute it. This creates **`dim_imsi_reference`** and inserts ~700 known subscriber records. The Lookup transformation will use this table to validate incoming IMSI values and retrieve their `subscriber_id`.

```sql
-- dim_imsi_reference structure
id            INT IDENTITY PRIMARY KEY
imsi          VARCHAR(9)
subscriber_id INT
```

> 📸 **Screenshot placeholder** — *SSMS showing dim_imsi_reference populated with rows*

---

## Step 2 — Understand the Source Data

The `Source Files/` directory contains three types of files the pipeline is designed to handle:

| File Pattern | Description | Notes |
|---|---|---|
| `01_clean_data.csv` | Fully clean rows | Happy-path test |
| `02_clean_data_with_null.csv` | Rows with intentional nulls and bad IDs | Tests error handling |
| `03_sample_data.csv` | Mixed: overflow IMSI, non-numeric ID (`1@`), letter event_type (`i`) | Tests truncation & conversion |
| `batch_01_file_*.csv` | Production-like batch files (20–200 rows each) | Main workload |
| `batch_02_file_*.csv` | Second batch set | Additional workload |

Known data quality issues the pipeline handles gracefully:

- Empty `imsi` field → row still flows through (lookup returns no match)
- Empty `cell` or `lac` field → integer conversion fails → row redirected to error table
- Non-numeric `id` like `1@` → conversion error → row redirected to error table
- Missing `event_ts` → datetime conversion fails → redirected to error table
- `imei` shorter than 14 characters → TAC/SNR derived columns return `"-99999"` sentinel

---

## Step 3 — Create the SSIS Project

1. Open **Visual Studio** → **New Project**
2. Choose **Integration Services Project** (under Business Intelligence)
3. Name it `ssis_telecom_DWH` and choose a local folder
4. Click **OK** — Visual Studio creates a blank `Package.dtsx` and opens the **Control Flow** canvas

> 📸 **Screenshot placeholder** — *Visual Studio with the blank SSIS project open, showing Control Flow tab*

---

## Step 4 — Configure the Foreach Loop Container

The loop container iterates over every `.csv` file in the source folder automatically.

### 4a. Add the container

Drag a **Foreach Loop Container** from the SSIS Toolbox onto the Control Flow canvas.

### 4b. Configure the Enumerator

Double-click the container → **Collection** tab:

| Setting | Value |
|---|---|
| Enumerator | Foreach File Enumerator |
| Folder | `C:\...\Source Files` (your absolute path) |
| Files | `*.csv` |
| Retrieve file name | **Fully qualified** (returns full path) |

> 📸 **Screenshot placeholder** — *Foreach Loop Editor — Collection tab showing file enumerator settings*

### 4c. Create package-level variables

Go to **SSIS → Variables** and create these four variables:

| Variable | Type | Value / Expression |
|---|---|---|
| `User::filepath` | String | `C:\...\Source Files\` |
| `User::fileExt` | String | `.csv` |
| `User::filename` | String | *(empty — mapped from the loop)* |
| `User::Fullpath` | String | **Expression:** `@[User::filepath] + @[User::filename] + @[User::fileExt]` |

Make `Fullpath` an **expression variable** (set `EvaluateAsExpression = True`).

> 📸 **Screenshot placeholder** — *SSIS Variables window showing all four variables*

### 4d. Map the loop output to a variable

Back in the Foreach Loop Editor → **Variable Mappings** tab:

- Add `User::filename` at **Index 0**

This writes each discovered filename (without path) into the variable on each loop iteration, which `Fullpath` then assembles into the complete file path.

> 📸 **Screenshot placeholder** — *Variable Mappings tab with User::filename mapped at index 0*

---

## Step 5 — Configure the Flat File Connection Manager

### 5a. Create the connection manager

Right-click the **Connection Managers** area → **New Flat File Connection**:

| Setting | Value |
|---|---|
| Connection manager name | `Flat File Connection Manager` |
| File name | Browse to any one of your `.csv` files (used as the schema template) |
| Locale | English (United States) |
| Code page | 1252 |
| Format | **Delimited** |
| Column names in first data row | ✅ Checked |

### 5b. Configure columns

On the **Columns** page, set the **column delimiter** to **Pipe `|`** and row delimiter to **`{CR}`**.

On the **Advanced** page, set the data types for each column:

| Column | SSIS Data Type | Notes |
|---|---|---|
| `id` | `DT_I4` (four-byte integer) | IgnoreFailure on error — non-numeric IDs get redirected |
| `imsi` | `DT_STR`, length 9 | |
| `imei` | `DT_STR`, length 14 | |
| `cell` | `DT_I4` | IgnoreFailure |
| `lac` | `DT_I4` | IgnoreFailure |
| `event_type` | `DT_STR`, length 1 | |
| `event_ts` | `DT_DATE` | IgnoreFailure |

> 📸 **Screenshot placeholder** — *Flat File Connection Manager — Advanced tab showing column data types*

### 5c. Bind the connection string to the variable

Right-click the connection manager → **Properties** → find **Expressions** → add:

| Property | Expression |
|---|---|
| `ConnectionString` | `@[User::Fullpath]` |

This makes the connection dynamically point to whichever file the loop currently holds.

> 📸 **Screenshot placeholder** — *Property Expressions Editor binding ConnectionString to @[User::Fullpath]*

---

## Step 6 — Build the Data Flow Task

Double-click the **Foreach Loop Container** to enter it, then drag a **Data Flow Task** onto the inner canvas.

Double-click the Data Flow Task to open the **Data Flow** tab. You will build the pipeline shown below by dragging components from the SSIS Toolbox:

```
[Flat File Source]
       ↓
  [Lookup]
       ↓ (Match Output)
[Derived Column]
       ↓
[Fact_Transaction OLE DB Destination]
       ↓ (Error Output)
[error_destination_output OLE DB Destination]
```

> 📸 **Screenshot placeholder** — *Completed Data Flow canvas with all five components connected by green/red arrows*

---

## Step 7 — Configure the Lookup Transformation

The Lookup validates whether an incoming IMSI is a known subscriber.

### 7a. Add and connect

Drag a **Lookup** transformation onto the Data Flow canvas. Connect the **Flat File Source output** (green arrow) to the Lookup input.

### 7b. General settings

Double-click the Lookup → **General** tab:

| Setting | Value |
|---|---|
| Cache mode | **No cache** (full cache may not fit into 25 MB default) |
| Connection type | OLE DB Connection Manager |
| No match behavior | **Redirect rows to no match output** ← critical setting |

### 7c. Connection

On the **Connection** tab, select your SQL Server connection manager and enter:

```sql
select * from [dbo].[dim_imsi_reference]
```

### 7d. Columns

On the **Columns** tab:

- Drag `imsi` (from the left, incoming data) to `imsi` (on the right, reference table) — this is the join key
- Check the **`subscriber_id`** checkbox to add it as an output column

> 📸 **Screenshot placeholder** — *Lookup Transformation Editor — Columns tab showing imsi join and subscriber_id selected*

---

## Step 8 — Configure the Derived Column Transformation

This transformation does two things: safely extracts TAC and SNR from the IMEI, and applies a null-safe default to `subscriber_id`.

### 8a. Add and connect

Drag a **Derived Column** transformation. Connect the **Lookup Match Output** (green arrow) to it.

### 8b. Expressions

Double-click the Derived Column and add three rows:

**Row 1 — Overwrite `subscriber_id`** (handle null when lookup matched but returned null):

| Derived Column Name | Derived Column | Expression |
|---|---|---|
| `subscriber_id` | \<replace 'subscriber_id'\> | `ISNULL(subscriber_id) ? -99999 : subscriber_id` |

**Row 2 — New column `TAC`** (first 8 digits of IMEI):

| Derived Column Name | Derived Column | Expression |
|---|---|---|
| `TAC` | \<add as new column\> | `(DT_STR,8,1252)(((ISNULL(imei) \|\| LEN(imei) < 14) ? "-99999" : SUBSTRING(imei,1,8)))` |

**Row 3 — New column `SNR`** (digits 9–14 of IMEI):

| Derived Column Name | Derived Column | Expression |
|---|---|---|
| `SNR` | \<add as new column\> | `(DT_STR,6,1252)(((ISNULL(imei) \|\| LEN(imei) < 14) ? "-99999" : SUBSTRING(imei,9,14)))` |

> 📸 **Screenshot placeholder** — *Derived Column Transformation Editor showing all three expression rows*

---

## Step 9 — Configure OLE DB Destinations

### 9a. Fact_Transaction (happy path)

Drag an **OLE DB Destination**, name it `Fact_Transaction`. Connect the **Derived Column output** (green arrow) to it.

Double-click → configure:

| Setting | Value |
|---|---|
| Connection manager | `OMAR\SQLEXPRESS.SSIS_Telecom_DB` |
| Table | `[dbo].[fact_transaction]` |
| Access mode | **Table or view — fast load** |
| Error handling on input | **Redirect row** (so insert failures go to error output) |

On the **Mappings** tab, map each incoming column to its destination column. Note that the source `id` column maps to `transaction_id` (the fact table's `id` column is an identity/surrogate key).

> 📸 **Screenshot placeholder** — *OLE DB Destination Mappings tab for fact_transaction*

### 9b. error_destination_output (error path)

Drag a second **OLE DB Destination**, name it `error_destination_output`. Connect the **Fact_Transaction Error Output** (red arrow) to it.

| Setting | Value |
|---|---|
| Table | `[dbo].[error_destination_output]` |
| Error handling | Fail component (we always want errors captured here) |

Map the same columns plus the two special error columns:

| Source | Destination |
|---|---|
| `ErrorCode` | `ErrorCode` |
| `ErrorColumn` | `ErrorColumn` |

> 📸 **Screenshot placeholder** — *Red error path arrow connecting Fact_Transaction to error_destination_output*

---

## Step 10 — Add the File System Task (Archive)

After the Data Flow Task succeeds, move the processed file to the `Processed Files/` folder so it won't be picked up again on the next run.

### 10a. Add the task

Inside the Foreach Loop Container (on the Control Flow canvas), drag a **File System Task** below the Data Flow Task.

### 10b. Create a destination variable

Inside the Foreach Loop Container's **Variables**, add:

| Variable | Type | Value |
|---|---|---|
| `User::destinationPath` | String | `C:\...\Processed Files` |

### 10c. Configure the task

Double-click the File System Task:

| Setting | Value |
|---|---|
| Operation | **Move file** |
| IsSourcePathVariable | True |
| SourceVariable | `User::Fullpath` |
| IsDestinationPathVariable | True |
| DestinationVariable | `User::destinationPath` |

### 10d. Connect with a precedence constraint

Draw a green success arrow from the **Data Flow Task** to the **File System Task**. This ensures the file only moves after the data has been loaded successfully.

> 📸 **Screenshot placeholder** — *Control Flow showing Data Flow Task → File System Task with green constraint arrow*

---

## Step 11 — Run and Validate

### 11a. Execute the package

Press **F5** or click **Start** in Visual Studio. The Foreach Loop will iterate through every `.csv` in `Source Files/`, and for each file:

1. Read rows via the Flat File Source
2. Validate IMSI against `dim_imsi_reference`
3. Derive TAC, SNR, and null-safe subscriber_id
4. Insert valid rows into `fact_transaction`
5. Redirect failed inserts to `error_destination_output`
6. Move the source file to `Processed Files/`

All components should turn **green**. Yellow warnings on the Flat File Source are expected (they indicate rows that were redirected due to conversion errors — this is intended behaviour).

> 📸 **Screenshot placeholder** — *SSIS package running, all tasks green, showing row counts on Data Flow arrows*

### 11b. Query the results

Open SSMS and run:

```sql
USE SSIS_Telecom_DB;

-- Check successfully loaded records
SELECT COUNT(*) AS loaded_rows FROM fact_transaction;
SELECT TOP 10 * FROM fact_transaction ORDER BY id DESC;

-- Check rejected / error rows
SELECT COUNT(*) AS error_rows FROM error_destination_output;
SELECT TOP 10 * FROM error_destination_output;

-- View the dimension table
SELECT TOP 10 * FROM dim_imsi_reference;
```

> 📸 **Screenshot placeholder** — *SSMS query results showing rows in both fact_transaction and error_destination_output*

---

## 🛡️ Error Handling Strategy

| Error Type | Example | Handling |
|---|---|---|
| Non-numeric `id` (e.g., `1@`, `text`) | `02_clean_data_with_null.csv` | `IgnoreFailure` on Flat File Source — row continues with null id |
| Missing `cell`, `lac`, `event_ts` | Empty field in source | `IgnoreFailure` — null value passes through |
| IMSI not found in dim table | New/unknown subscriber | Lookup routes to **No Match Output** (currently discarded; can be wired to error table) |
| Missing or short IMEI | IMEI field is null or < 14 chars | Derived Column returns `"-99999"` sentinel for TAC/SNR |
| Insert failure into fact table | NOT NULL constraint violation | `RedirectRow` — sent to `error_destination_output` with error code |

---

## 📂 Repository Structure

```
📦 ssis_telecom_DWH/
 ┣ 📂 Source Files/              ← Drop new CSV batches here
 │   ┣ 01_clean_data.csv
 │   ┣ 02_clean_data_with_null.csv
 │   ┣ 03_sample_data.csv
 │   ┣ batch_01_file_01.csv … batch_01_file_05.csv
 │   ┗ batch_02_file_01.csv … batch_02_file_05.csv
 ┣ 📂 Processed Files/           ← Files move here after successful load
 ┣ 📂 SQL Queries/
 │   ┣ Create database.sql       ← Run first: creates DB, fact & error tables
 │   ┗ Create dim imsi.sql       ← Run second: creates & seeds dim_imsi_reference
 ┣ 📂 ssis_telecom_DWH/          ← Visual Studio SSIS project
 │   ┣ Package.dtsx              ← Main ETL package
 │   ┣ Project.params
 │   ┗ ssis_telecom_DWH.dtproj
 ┣ 📂 images/                    ← Screenshots for this README
 ┗ 📜 README.md
```

---

## 🔧 Troubleshooting

**The Foreach Loop picks up no files**
- Verify the folder path in the Foreach Loop Enumerator matches where your CSVs actually live
- Confirm the `FileSpec` is set to `*.csv`
- Check that `FileNameRetrievalType` is set to **Name only** (not fully qualified — `Fullpath` variable assembles the full path)

**"Column delimiter not found" or wrong row count**
- Open the Flat File Connection Manager, go to **Columns**, and confirm the delimiter is `|` (pipe), not comma or tab
- Confirm row delimiter is `{CR}` or `{CR}{LF}` matching your file line endings

**Lookup transformation shows 0 matches**
- The `imsi` column length must be `VARCHAR(9)` on both sides of the join — check the flat file column and the dim table column are the same type/length
- Run `SELECT * FROM dim_imsi_reference WHERE imsi = '257865245'` to confirm data was seeded

**All rows go to the error table**
- Verify the `event_ts` date format in the CSV (`dd/MM/yyyy HH:mm`) matches what the Flat File Source expects — this is the most common insertion failure cause
- Check that `fact_transaction.imsi NOT NULL` constraint isn't triggered — rows with null IMSI should still be allowed through by the `IgnoreFailure` setting

**File System Task fails**
- Ensure the `Processed Files/` directory exists before running
- Confirm the `User::destinationPath` variable value ends without a trailing slash

**Package fails on second run (files already moved)**
- This is expected behaviour — once a file is in `Processed Files/`, the loop won't see it again. To re-test, copy files back to `Source Files/`

---

*Built with SQL Server Integration Services (SSIS) · Visual Studio Data Tools · SQL Server Express*
