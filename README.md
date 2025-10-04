# AdventureWorks — Azure Data Engineering Project

> End-to-end demo of an Azure **medallion** data platform: ingest (ADF/HTTP ➜ ADLS Gen2 **Bronze**), transform (Databricks ➜ ADLS Gen2 **Silver**), serve (Synapse **Gold**), and report (Power BI).

---

## 🔎 Project Overview

This repository showcases a reproducible data engineering pipeline built on Azure.  
It ingests AdventureWorks sample data from HTTP sources, lands it in an Azure Data Lake, transforms it with Databricks, models it for analytics in Synapse, and publishes dashboards in Power BI.

---

## 🧱 Architecture

## Architecture

![Architecture](architecture.png)



| Stage                | Service             | Layer  | Purpose                                          |
|----------------------|--------------------|--------|--------------------------------------------------|
| **Data Source**       | HTTP                | –      | AdventureWorks CSVs                              |
| **Ingestion**         | Azure Data Factory  | –      | Pull data from HTTP into ADLS Gen2 (raw)         |
| **Raw Store (Bronze)**| Azure Data Lake Gen2| Bronze | Landing area, unprocessed data                   |
| **Transformation**    | Azure Databricks    | Silver | Clean and model raw data into curated tables     |
| **Serving**           | Azure Synapse       | Gold   | Star schema for analytics and BI                 |
| **Reporting**         | Power BI            | –      | Dashboards and KPIs                              |

---

## 📦 Dataset Overview

### Fact tables
- **Sales (2015/2016/2017)**  
  Columns: `OrderDate`, `StockDate`, `OrderNumber`, `ProductKey`, `CustomerKey`, `TerritoryKey`, `OrderLineItem`, `OrderQuantity`
- **Returns**  
  Columns: `ReturnDate`, `TerritoryKey`, `ProductKey`, `ReturnQuantity`

### Dimension tables
- **Products**  
  Columns: `ProductKey`, `ProductSubcategoryKey`, `ProductSKU`, `ProductName`, `ModelName`, `ProductDescription`, `ProductColor`, `ProductSize`, `ProductStyle`, `ProductCost`, `ProductPrice`
- **Product Subcategories**  
  Columns: `ProductSubcategoryKey`, `SubcategoryName`, `ProductCategoryKey`
- **Product Categories**  
  Columns: `ProductCategoryKey`, `CategoryName`
- **Customers**  
  Columns: `CustomerKey`, `Prefix`, `FirstName`, `LastName`, `BirthDate`, `MaritalStatus`, `Gender`, `EmailAddress`, `AnnualIncome`, `TotalChildren`, `EducationLevel`, `Occupation`, `HomeOwner`
- **Territories**  
  Columns: `SalesTerritoryKey`, `Region`, `Country`, `Continent`
- **Calendar**  
  Columns: `Date`

---

## ⭐ Target Data Model

- **Fact tables:** Sales (by order line), Returns (by product × territory × return date)  
- **Dimension tables:** Products, Subcategories, Categories, Customers, Territories, Calendar  

**Joins:**  
- Sales ↔ Products on `ProductKey`  
- Sales ↔ Customers on `CustomerKey`  
- Sales ↔ Territories on `TerritoryKey = SalesTerritoryKey`  
- Sales ↔ Calendar on `OrderDate = Date`  
- Returns ↔ Products/Territories/Calendar similarly  

This forms a classic **star schema** to support analytics.

---

## 🧭 Medallion Plan

- **Bronze (Raw):** Store CSVs exactly as received in ADLS Gen2.  
- **Silver (Transformed):** Enforce schema & types, add date parts, validate foreign keys.  
- **Gold (Serving):** Star schema in Synapse for Power BI.

---

## 🛠️ Tech Stack

- **Azure Data Factory** — ingestion (HTTP → ADLS Gen2 Bronze)  
- **Azure Databricks** — PySpark transformations (Bronze → Silver)  
- **Azure Synapse** — serving layer (Gold)  
- **Power BI** — dashboards & reporting  
- **ADLS Gen2** — data lake storage  

---

## ⚙️ Step-by-Step Implementation  

### 1. Resource Setup  
- **Create Resource Group** – logical container for all resources.  
- **Create Storage Account (Resource 1)** – use **Locally Redundant Storage (LRS)**.  
- **Enable Hierarchical Namespace** – converts blob storage into **Data Lake Gen2** (required for folder-like structures and table storage).  
- **Create Azure Data Factory (Resource 2)** – for data ingestion pipelines.  
- **Define Medallion Containers** – within the Data Lake, create three containers:  
  - `bronze` → raw, unprocessed data  
  - `silver` → transformed, curated data  
  - `gold` → serving layer for analytics  

---

### 2. Data Ingestion (ADF → Bronze)  
- Use **Azure Data Factory Pipelines** to ingest data from GitHub (source) into ADLS Gen2 (destination).  
- Required **Linked Services**:  
  - **HTTP** (base URL from GitHub)  
  - **Azure Data Lake Gen2** (destination)  
- **Pipeline Activities**:  
  1. **Copy Activity** → moves data from HTTP source to ADLS Gen2 Bronze container.  
  2. **Source Configuration** → specify the relative URL of the raw CSV.  
  3. **Sink Configuration** → map to Bronze container in ADLS Gen2.  

---

### 3. Static vs. Dynamic Pipelines  
- **Static Pipeline:** Current setup ingests a single file at a time.  
- **Dynamic Pipeline (Recommended in Real Projects):**  
  - Uses **Iteration & Conditionals** (e.g., *For Each* activity).  
  - Dynamically parameterizes file names/paths.  
  - Enables ingestion of multiple files in a single automated pipeline run.  

---

### 4. Creating Dynamic Pipelines in ADF  

To make ingestion scalable across multiple files without hardcoding:  

1. **Parameterization**  
   - Create **one parameter** for the HTTP source file (CSV).  
   - For the ADLS Gen2 sink, define **two parameters**: one for the folder (directory) and one for the file.  
   - In total, you will use **3 parameters** (1 source + 2 sink).  

2. **For Each Activity**  
   - Add a loop (`ForEach`) to iterate through files in your GitHub repo.  
   - The iteration list comes from a **JSON array/dictionary** of relative file URLs and dataset metadata.  

3. **Lookup Activity**  
   - Use a **Lookup activity** to read the JSON file that contains the dataset specifications.  
   - The `ForEach` activity references this Lookup output and passes values into parameters.  

4. **Embedding the Copy Activity**  
   - Place the **Dynamic Copy activity** inside the `ForEach`.  
   - This way, each iteration dynamically pulls the right file from GitHub and lands it in the correct ADLS Gen2 folder.  

📌 *Tip:* This approach lets you add new CSVs to your JSON spec without modifying the pipeline itself—making it extensible and production-friendly.  
📸 Add supporting screenshots 


## 📸 Embedding DynamicCopy & Parameters in ForEach Activity

**1. Embedding DynamicCopy into ForEach activity**  
![embedding1](embedding_dynamiccopy_in_foreach_activity.png)

**2. Embedding Parameters within ForEach as Source**  
![embedding2](embedding_parameters_in_forEach_Source.png)

**3. Embedding Parameters within ForEach as Sink/Destination**  
![embedding2](embedding_parameters_in_forEach_Sink.png)



---

✅ With this, **Phase 1 (Bronze layer)** is complete.  
Next, we move into **Phase 2 (Silver layer)**, where transformations are performed using **Azure Databricks**.  


---

### 5. Silver Phase — Transformation with Azure Databricks  

After completing the Bronze ingestion, the next step is to clean and transform the data in **Azure Databricks**.  

- **Databricks Setup**  
  - Create a compute cluster within Azure Databricks.  
  - Connect Databricks Notebooks to **Azure Data Factory (ADF)**.  
  - Note: This requires credentials and secure communication between Databricks and ADF.  

- **Authentication & Access**  
  1. Create a **Microsoft Entra ID key (certificate)** that holds the authentication credentials.  
  2. Generate a **secret/key** for your Entra ID.  
  3. Assign the role **Storage Blob Contributor** to your storage account in Azure.  

- **Data Loading**  
  - Use official Databricks documentation to load data from ADLS Gen2 into Databricks.  
  - Perform schema enforcement, cleansing, and enrichment.  
  - Add date parts, enforce data types, and validate relationships.  

- **Data Writing**  
  - Save transformed datasets in **Parquet** format into the **Silver container** in ADLS Gen2.  
  - Once all files are transformed and saved, your Silver layer is complete.  

> 📓 *Refer to the provided `.ipynb` notebooks for transformation code.  
> Advanced transformations and business rules are further handled in **Power BI** using Table View and **DAX** for flexible analysis and visualization.*  

---

### 6. Gold Phase — Serving with Azure Synapse  

The **Gold Layer** is the serving/consumption layer. Here, curated and business-ready datasets are exposed to BI tools and end-users.  

**Step 1: Synapse Setup**  
- Create an **Azure Synapse Analytics workspace**.  
- Configure a **serverless SQL database**.  

**Step 2: Permissions**  
- Assign **Storage Blob Contributor** role to Synapse and your user identity.  

**Step 3: Query Silver Data**  
- Use **`OPENROWSET()`** to query curated data directly from ADLS Silver.  
- This allows schema validation and lightweight exploration before publishing.  

**Step 4: Create Views**  
- Build logical **views** for each Silver dataset (Sales, Customers, Products, Territories, Returns, Calendar, etc.).  

**Step 5: Create External Tables**  
- Define:  
  - **Credentials** (managed identity or keys).  
  - **External Data Sources** (point to Silver and Gold containers).  
  - **External File Formats** (Parquet, CSV, JSON, with compression).  
- Promote datasets into **external tables** aligned with fact and dimension roles.  

**Step 6: Data Modeling**  
- Organize into a **star schema** with fact tables (Sales, Returns) and dimension tables (Products, Customers, Territories, Calendar, Product Subcategories).  
- Optimize with partitioning/indexing for efficient queries.  

**Step 7: Consumption**  
- Power BI connects to Synapse Gold tables for dashboards and KPIs.  

**References:**  
- [Create Database Master Key](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-master-key-transact-sql?view=sql-server-ver17)  
- [Create Database Scoped Credential](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-database-scoped-credential-transact-sql?view=sql-server-ver17)  
- [Create External File Format](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-external-file-format-transact-sql?view=sql-server-ver17&tabs=delimited)  

---

## 🔄 Medallion Workflow Recap  

1. **Bronze Layer (Raw Store):**  
   - Direct ingestion from GitHub into ADLS Gen2.  
   - Stores CSVs in raw, unmodified form.  

2. **Silver Layer (Transformation):**  
   - Databricks cleans, validates schema, and enriches the data.  
   - Adds date parts, enforces types, ensures referential integrity.  

3. **Gold Layer (Serving):**  
   - Curated star schema tables published into Synapse.  
   - Optimized for BI consumption.  

---

## 🚀 Roadmap (initial)

- [x] Defined resource setup (RG, ADLS, ADF)  
- [x] Bronze ingestion pipeline created (static)  
- [x] Bronze ingestion pipeline upgraded (dynamic with parameters + ForEach)  
- [x] Databricks notebooks for Silver transformations  
- [x] Synapse schema deployment (Gold)  
- [x] Power BI dashboard integration  


---

## 📄 License & Attribution

Dataset adapted from **AdventureWorks** sample data. Use for learning and portfolio purposes.

---
Contains files for personal projects
