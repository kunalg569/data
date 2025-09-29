# AdventureWorks — Azure Data Engineering Project

> End-to-end demo of an Azure **medallion** data platform: ingest (ADF/HTTP ➜ ADLS Gen2 **Bronze**), transform (Databricks ➜ ADLS Gen2 **Silver**), serve (Synapse **Gold**), and report (Power BI).

---

## 🔎 Project Overview

This repository showcases a reproducible data engineering pipeline built on Azure.  
It ingests AdventureWorks sample data from HTTP sources, lands it in an Azure Data Lake, transforms it with Databricks, models it for analytics in Synapse, and publishes dashboards in Power BI.

---

## 🧱 Architecture

![Architecture](./docs/architecture.png) <!-- Replace with the actual path of your architecture image -->

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

## 🚀 Roadmap (initial)

- [ ] Land all CSVs to Bronze (ADF HTTP → ADLS Gen2)  
- [ ] Create Databricks notebooks to build Silver tables  
- [ ] Publish star schema in Synapse (Gold)  
- [ ] Build Power BI dashboards (Sales trends, Return rates, Geo breakdown)

---

## 📄 License & Attribution

Dataset adapted from **AdventureWorks** sample data. Use for learning and portfolio purposes.

---
Contains files for personal projects
