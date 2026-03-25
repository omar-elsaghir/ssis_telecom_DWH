# 📡 Telecom Data Warehouse ETL Pipeline

## 📝 Overview
This repository contains an end-to-end SQL Server Integration Services (SSIS) ETL pipeline designed for a Telecommunications Data Warehouse. The package automates the ingestion of daily and batch Call Detail Records (CDRs) and transaction logs, cleanses the data, performs lookups against dimension tables, and securely loads the validated records into a Fact Table. 

The project emphasizes robust error handling, ensuring that malformed data (like missing IDs or type conversion failures) does not break the pipeline, but is instead redirected to an audit table for review.

## 🛠️ Tech Stack
* **ETL Tool:** SQL Server Integration Services (SSIS) / Visual Studio Data Tools
* **Database:** Microsoft SQL Server (T-SQL)
* **Source Data:** CSV Flat Files (Telecom batch data)
* **Version Control:** Git & GitHub

## 🏗️ ETL Architecture & Data Flow

The core of the SSIS package is built around a **Foreach Loop Container** that iterates through a designated `Source Files` directory, picking up new `.csv` batch files and processing them through the following Data Flow:

1.  **Flat File Source:** Reads the incoming CSV data. Configured to handle large telecom identifiers (using `DT_I8` / BigInt) to prevent data overflow errors.
2.  **Lookup Transformation:** Validates incoming records against existing Dimension tables (e.g., IMSI or cell tower data) using a cached lookup.
3.  **Derived Column:** Cleanses strings and handles whitespace formatting before database insertion.
4.  **OLE DB Destinations:** * *Match Output:* Successfully validated rows are inserted into `Fact_Transaction`.
    * *No Match / Error Output:* Rows failing conversion or lookup are cleanly redirected to `error_destination_output`.

### Data Flow Design
![SSIS Data Flow Design](images/Screenshot%202026-03-25%20172423.png)
*Above: The core Data Flow showing the Lookup, Derived Column, and OLE DB Destination components.*

### Error Handling Configuration
![Error Output Configuration](images/Screenshot%202026-03-25%20175300.png)
*Above: Configuring component error outputs to redirect rows instead of failing the package.*

### Successful Execution
![SSIS Package Success](images/Screenshot%202026-03-25%20180623.png)
*Above: A successful run of the Foreach Loop Container, processing multiple batches and safely redirecting malformed rows.*

---

## 🚀 Step-by-Step Implementation Guide

### Step 1: Database Setup
Before running the SSIS package, the target data warehouse schema must be created.
1. Execute `Create database.sql` to initialize the DWH environment.
2. Execute `Create dim imsi.sql` to build the required dimension tables.
3. Ensure the `error_destination_output` table is created to catch rejected rows.

### Step 2: SSIS Control Flow Configuration
1. Add a **Foreach Loop Container** to the Control Flow.
2. Configure the enumerator to scan the `Source Files` folder for `*.csv`.
3. Map the File Path to a variable (e.g., `User::CurrentFilePath`) so the Flat File Connection Manager updates dynamically for each file.

---

## 💡 Troubleshooting & Development Notes

During the development of this pipeline, several environment and version control challenges were resolved:

### 1. Visual Studio UI Rendering (White Text Bug)
An issue was encountered where Data Flow Path labels (like row counts) rendered as unreadable white text.
**Fix applied:**
* Navigated to **Tools > Options > Environment > Fonts and Colors**.
* Selected **Business Intelligence Designers** from the dropdown.
* Changed the **Data Flow Path Label** foreground to Black.

![Visual Studio UI Fix](images/Screenshot%202026-03-25%20180529.png)

### 2. Git Merge Conflicts & SSIS Files
SSIS projects generate many background XML and caching files (`.params`, `.database`) that can cause severe Git conflicts when pulling or merging branches. 

![Git Commit History](images/Screenshot%202026-03-25%20192341.png)
*Analyzing the divergent branch history prior to resolving conflicts.*

![Git Conflict Resolution](images/Screenshot%202026-03-25%20192347.png)
*Resolving unmerged changes by selecting 'Keep Local' for SSIS parameter files to maintain the local configuration.*

### 3. The ".vs Folder" Permission Denied Error
Visual Studio actively locks temporary files inside the hidden `.vs` folder. Trying to commit or push these via Git resulted in a fatal `Permission denied` error.

![Git Permission Error](images/Screenshot%202026-03-25%20192354.png)

**Resolution:**
1. Closed Visual Studio entirely to release the file locks.
2. Deleted the hidden `.vs` folder in the project directory.
3. Re-opened Visual Studio, committed, and pushed successfully.

**Best Practice Applied:** A `.gitignore` file has been implemented in this repository to automatically ignore `.vs/`, `*.user`, and `/bin/` directories, preventing future tracking conflicts.

---

## 📂 Repository Structure
```text
📦 ssis_telecom_DWH
 ┣ 📂 Source Files/         # Incoming raw CSV batches
 ┣ 📂 Processed Files/      # Archive of completed batches
 ┣ 📂 SQL Queries/          # DDL scripts for DWH setup
 ┣ 📂 ssis_telecom_DWH/     # Visual Studio SSIS Project files
 ┃ ┣ 📜 Package.dtsx        # Main ETL Package
 ┃ ┗ 📜 Project.params      # Project-level parameters
 ┣ 📜 .gitignore            # Git ignore rules for SSIS
 ┗ 📜 README.md             # Project documentation
