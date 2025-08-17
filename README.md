# CDI Bonus Calculation Data Product

## 1. Overview

This document outlines the design, implementation, and operational procedures for the CDI Bonus Calculation data product. The project's primary goal is to calculate daily interest (based on the CDI rate) for eligible user wallet balances and produce a daily payout dataset.

The solution is implemented as a Databricks notebook using PySpark. It leverages **Unity Catalog** for data governance and follows the Medallion architecture for data processing. The design ensures the pipeline is reliable, auditable, and scalable.

## 2. Instructions for Installation, Testing, and Execution

### 2.1. Prerequisites

* **Databricks Environment:** An active Databricks workspace. This solution is fully compatible with the **Databricks Community Edition (Free Version)**.
* **Unity Catalog:** Unity Catalog must be enabled to manage data assets (tables and volumes).
* **Bronze Layer Data:**
    * **Transaction Files:** For this project, the raw transaction Parquet files were manually uploaded to a **Databricks Volume**. The pipeline is configured to read from the path: `/Volumes/rp/bronze/transactions/`.
    * **Calendar Table:** A calendar dimension table must exist in Unity Catalog as `rp.bronze.dim_calendario`. It is important to note that this table was generated using a specialized library that includes all national holidays and, crucially, **banking holidays based on the official FEBRABAN calendar**. This ensures maximum accuracy for financial calculations.
* **Permissions:** The cluster/user must have `READ` permissions on the Bronze layer assets and `WRITE` permissions on the `rp.silver` and `rp.gold` schemas within Unity Catalog.

### 2.2. Configuration

The Databricks notebook is parameterized. Before execution, ensure the variables in the second cell of the notebook are correctly configured:

* `bronze_transactions_path`: Path to the raw transaction files in the Volume.
* `bronze_calendar_table`: Full name of the FEBRABAN calendar table.
* `silver_eligible_balances_table`: Full name for the intermediate Silver table.
* `gold_payouts_table`: Full name for the final Gold output table.
* `processing_date_str`: For production, this should be configured as a Databricks Widget to allow scheduled runs for specific dates.

### 2.3. Execution

1.  **Manual Execution:** Open the notebook, set the `processing_date_str` variable, and run all cells.
2.  **Scheduled Job:** Create a Databricks Job to run the notebook daily. Use the job parameters to set the `processing_date` dynamically.

## 3. Explanation of Design Choices

### 3.1. Architectural Design: The Medallion Architecture with Unity Catalog

The solution is built on the Medallion Architecture, with all assets governed by **Unity Catalog**. This provides centralized access control, data discovery, and lineage across the entire pipeline.

* **Bronze Layer:** Holds raw data. We read transaction files directly from a Volume and the calendar table, ensuring a durable source of truth.
* **Silver Layer:** Creates the clean and conformed `daily_eligible_balances` table. This intermediate, modeled table is crucial for auditing the core business logic.
* **Gold Layer:** Contains the final, aggregated `cdi_daily_payouts` table, ready for business consumption by the downstream payment system.

### 3.2. Fulfilling Functional Requirements

* **Wallet History:** The pipeline processes the full transaction history to accurately calculate the opening balance for each user as of the last business day.
* **Interest Calculation Logic:**
    * The balance stability rule is efficiently implemented by identifying and excluding users with transactions since the last business day using a `left_anti` join.
    * The `$100` balance threshold is applied using a `.filter()` operation.
* **Time Frame & Payout:** The use of the comprehensive `dim_calendario`, built from the **official FEBRABAN banking holiday calendar**, makes the logic extremely robust. The pipeline correctly identifies the last business day and calculates the number of days to accrue interest, accurately handling weekends and banking holidays.

### 3.3. Fulfilling Non-Functional Requirements

* **Mission-Critical Reliability:**
    * **Atomicity & Idempotency:** Using **Delta Lake** tables in Unity Catalog ensures that writes are atomic. The `replaceWhere` option makes the job idempotent, allowing safe re-runs without duplicating or corrupting financial data.
* **Visibility and Auditing:** The multi-layered architecture in Unity Catalog provides clear data lineage. If a payout is questioned, one can easily trace the calculation from the final Gold table back through the Silver business rules to the original Bronze data.

## 4. Recommendations for Production Ingestion

The `gold.cdi_daily_payouts` table contains the data that needs to be loaded into the production transactional database. The recommended approach is a **Reverse ETL** pattern.

1.  **Decoupled Ingestion Service:** A separate, dedicated service should handle this process.
2.  **Read from Gold Table:** The service will query the Gold table for new payouts that have not yet been processed.
3.  **Transactional Database Writes:** For each payout, the service must perform the database write within an explicit transaction (`BEGIN`, `COMMIT`, `ROLLBACK`).
4.  **Error Handling and Reconciliation:** The service must have robust error handling and alerting. A daily reconciliation process should verify that totals in the Gold table match the totals loaded into the production database.

## 5. Compromises and Trade-offs

* **Interest Rate Source:** The CDI rates are currently simulated in a DataFrame. In production, this data would be ingested from a reliable financial data provider API.
* **Performance at Scale:** The current logic scans the full transaction history. For massive data volumes, a future optimization would be to maintain a daily balance snapshot table in the Silver layer to accelerate the calculation.
* **Full Test Coverage:** The provided notebook contains the core logic but lacks a formal suite of unit and integration tests, which would be required for a production-grade deployment.