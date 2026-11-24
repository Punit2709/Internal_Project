
# 🚀 ADF Pipeline Documentation

## 📌 Overview

This documentation outlines two core pipelines built using **Azure Data Factory (ADF)**:

1. **Run All Pipeline** – Triggers the full data flow process
2. **Remote ADLS ➡️ Local ADLS** – Transfers updated files based on metadata

---

## 1️⃣ Run All Pipeline

### 🧩 Purpose

The **Run All Pipeline** orchestrates the end-to-end data lifecycle by executing three sub-pipelines sequentially:

1. 🔄 **Data Ingestion**
2. 🔧 **Transformation**
3. 📤 **Data Dumping**

### 🛠️ Implementation

This pipeline uses **Execute Pipeline** activities to manage dependencies and execution flow.

📸 **Screenshots**:
![Run All 1](/images/adf/run_all_1.png)
![Run All 2](/images/adf/run_all_2.png)
![Run All 3](/images/adf/run_all_3.png)
![Run All 4](/images/adf/run_all_4.png)

---

## 2️⃣ Remote ADLS ➡️ Local ADLS Pipeline

### 🧩 Purpose

This pipeline copies only the **new or modified files** from a **Remote ADLS** container to a **Local ADLS** container using metadata-based filtering.

### 🧠 Working Logic

* A **pipeline variable** called `lastRunTime` stores the last execution time.
* **Metadata Activity** is used to list all files along with their **last modified time**.
* A **ForEach Activity** filters out only files modified after `lastRunTime`.
* At the end, `lastRunTime` is updated to current time using `@utcNow()`.

---

### 🛠️ Implementation Steps

#### 📍 Step 1: Metadata Activity

* **Source**: Remote ADLS
* **Sink**: Local ADLS
* Extracted Metadata Fields:

  * 🧾 `childItems` – Lists all files
  * 🕒 `lastModified` – Used to filter new/updated files

📸 Screenshots:
![Metadata Overview](/images/adf/ing_0.png)
![Metadata Config 1](/images/adf/ing_1.png)
![Metadata Config 2](/images/adf/ing_2.png)
![Metadata Config 3](/images/adf/ing_3.png)

---

#### 🔁 Step 2: ForEach Activity

Filters files based on:

* `lastRunTime`
* `@utcNow()`

Only files modified after `lastRunTime` are processed.

📸 Screenshots:
![ForEach 1](/images/adf/ing_4.png)
![ForEach 2](/images/adf/ing_5.png)
![ForEach 3](/images/adf/ing_6.png)
![ForEach 4](/images/adf/ing_7.png)
![ForEach 5](/images/adf/ing_8.png)
![ForEach 6](/images/adf/ing_9.png)

---

#### 🧮 Step 3: Set Variable Activity

* Updates the `lastRunTime` with current time
* Expression used: `@utcNow()`

📸 Screenshot:
![Set Variable](/images/adf/ing_10.png)

---

## 3️⃣ Data Transformation Pipeline

### 🧩 Purpose

This pipeline focuses on executing **Databricks Notebooks** to perform data transformation across the **Bronze, Silver, and Gold layers**.

---

### ⚙️ Pipeline Flow & Notebook Activities

This pipeline includes **4 key notebook activities** executed in sequence:

1. 🛠 **Preprocessing Notebook**

   * Sets up **ADLS-Databricks connection**
   * Mounts necessary storage locations
     📍 *Prepares the workspace for downstream tasks*

2. 🥉 **Bronze Layer Notebook**

   * Uses **Autoloader** to ingest raw data
   * Creates the `bronze_layer` table
     📍 *First layer of structured storage*

3. 🥈 **Silver Layer Notebook**

   * Transforms raw data into **normalized, deduplicated** dimension and fact tables
     📍 *Refined and business-readable data*

4. 🥇 **Gold Layer Notebook**

   * Calculates business **KPIs**, aggregations, and reporting outputs
     📍 *Final layer used for analytics and reporting*

---

### 🖼️ Implementation Screenshots

![📋 Pipeline Overview](/images/adf/ing_11.png)
![🧱 Notebook Start](/images/adf/dbx_1.png)

#### 🔧 Preprocessing & Bronze Layer

![Mounting & Setup](/images/adf/dbx_2.png)
![Bronze Step 1](/images/adf/dbx_3.png)
![Bronze Step 2](/images/adf/dbx_4.png)
![Bronze Step 3](/images/adf/dbx_5.png)
![Bronze Step 4](/images/adf/dbx_6.png)
![Bronze Step 5](/images/adf/dbx_7.png)

#### ⚙️ Silver Layer

![Silver Step 1](/images/adf/dbx_8.png)
![Silver Step 2](/images/adf/dbx_9.png)
![Silver Step 3](/images/adf/dbx_10.png)

#### 📊 Gold Layer

![Gold Step 1](/images/adf/dbx_11.png)
![Gold Step 2](/images/adf/dbx_12.png)
![Gold Step 3](/images/adf/dbx_13.png)

---

### 🔗 Databricks Link Service

This section defines the connection between **Azure Data Factory** and **Databricks**.

* Configures **Databricks Workspace URL**
* Specifies **Cluster ID** to be used
* Authenticated using a **Databricks Access Token**

📸 Screenshot:
![Databricks Link Service](/images/adf/dbx_14.png)

---

## 4️⃣ Data Dumping Pipeline

### 🧩 Purpose

This pipeline is responsible for transferring **KPI output data** (stored as Parquet files in a Databricks external location) to **Snowflake tables**.

It ensures:

* Only the **latest version** of each KPI table is picked
* Tables are dynamically created in Snowflake
* Data is efficiently loaded using a **staging area**

---

### ⚙️ Pipeline Flow

This pipeline includes the following core activities:

1. 📁 **Metadata Activity**

   * Retrieves folders (each representing a KPI table)

2. 🔁 **ForEach Activity**

   * Iterates through each KPI folder
   * Identifies the **latest parquet file**
   * Triggers **dynamic Snowflake table creation** and **data load**

---

### 🖼️ Implementation Screenshots

#### 🧾 Pipeline Overview

![Pipeline Overview](/images/adf/dd_1.png)

---

### 📂 Metadata Activity – Folder Level

* Scans the directory that holds **one folder per KPI**
* Retrieves structure to iterate further

![Metadata 1](/images/adf/dd_2.png)
![Metadata 2](/images/adf/dd_3.png)
![Metadata 3](/images/adf/dd_4.png)

---

### 🔁 ForEach Activity

* Loops through each folder (KPI table)
* Kicks off a sub-pipeline per folder

![ForEach 1](/images/adf/dd_5.png)
![ForEach 2](/images/adf/dd_6.png)
![ForEach 3](/images/adf/dd_7.png)

---

### 📌 Set Variable – Folder Name

* Stores the current KPI folder name (used in next steps)

![Set Folder Name](/images/adf/dd_8.png)

---

### 📂 Get Metadata – File Level

* Retrieves the list of Parquet files within a KPI folder
* Extracts file names and last modified timestamps

![File Metadata](/images/adf/dd_9.png)

---

### 🔍 Filter Activity – Latest File

* Filters files to select only the **most recent Parquet file**

![Filter Latest File](/images/adf/dd_10.png)

---

### 📝 Set Variable – Filename

* Captures the name of the filtered latest file

![Set Filename](/images/adf/dd_11.png)

---

### 🧹 Delete Activity – Clean Stage Area

* Clears the **Snowflake stage area** (ADLS Gen1) before loading
* Ensures no conflict with previous data

![Delete Stage 1](/images/adf/dd_12.png)
![Delete Stage 2](/images/adf/dd_13.png)
![Delete Stage 3](/images/adf/dd_14.png)

---

### 📄 Lookup & Script Activity – Snowflake Table Creation

* Uses **Lookup** to fetch schema dynamically
* **Script Activity** creates table in Snowflake before data load

![Lookup](/images/adf/dd_15.png)
![Script 1](/images/adf/dd_16.png)
![Script 2](/images/adf/dd_17.png)

---

### 📥 Copy Into Activity – Load to Snowflake

* Performs the **actual data copy** from ADLS to Snowflake
* Utilizes an **intermediate stage (ADLS Gen1)** for Snowflake compatibility

![Copy Data 1](/images/adf/dd_18.png)
![Copy Data 2](/images/adf/dd_19.png)

---

### ✅ Summary

| Component          | Purpose                                |
| ------------------ | -------------------------------------- |
| Metadata Activity  | Retrieves KPI folders and file details |
| ForEach Activity   | Iterates through KPI folders           |
| Set Variable       | Tracks folder and latest file name     |
| Filter Activity    | Picks latest Parquet file              |
| Delete Activity    | Cleans Snowflake stage area            |
| Script Activity    | Dynamically creates Snowflake tables   |
| Copy Data Activity | Loads data using staging via ADLS Gen1 |

---