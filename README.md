# SQL Data Warehouse & Analytics Project 🚀

This is the **SQL Data Warehouse and Analytics Project**! This repository showcases a comprehensive end-to-end data engineering solution—transforming raw, fragmented data into actionable business intelligence using **Microsoft SQL Server**.

## Data Architecture
The project follows the **Medallion Architecture**, ensuring a clean and scalable data pipeline:

* **🥉 Bronze Layer (Raw):** Stores the source data in its original format. I used Bulk Insert operations to bring in CSV data from ERP and CRM systems.
* **🥈 Silver Layer (Cleansed):** Data is scrubbed for inconsistencies. This includes handling `NULL` values, standardizing date formats, and removing duplicates to ensure a "Single Source of Truth."
* **🥇 Gold Layer (Business):** Data is modeled into a **Star Schema** (Fact and Dimension tables). This layer is optimized for high-performance analytical queries and BI reporting.

## Objectives
* **ETL Automation:** Developed T-SQL Stored Procedures to automate data movement across layers.
* **Data Quality:** Implemented logic to map disparate values (e.g., standardizing gender codes and product categories).
* **Analytical Readiness:** Designed the schema to easily answer questions regarding customer lifetime value, product popularity, and sales growth.

## Technology Stack
* **Engine:** Microsoft SQL Server
* **Language:** T-SQL (Transact-SQL)
* **Modeling:** Star Schema (Fact & Dimensions)
* **Tooling:** SQL Server Management Studio (SSMS)

## Repository Structure
* **`/datasets`**: Source CSV files used for ingestion.
* **`/scripts`**: Organized SQL scripts for table creation (`DDL`) and data loading (`Stored Procedures`).
* **`/docs`**: Data Catalog describing the data in all layers.
