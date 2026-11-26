# 🏥 MediLake – Healthcare Data Lakehouse on Azure

An end-to-end data engineering project that ingests healthcare data (patients, hospitals, lab tests) from multiple sources, processes it through a **Bronze → Silver → Gold medallion architecture** on **Azure Databricks**, orchestrates it with **Azure Data Factory**, sends run-status alerts through **Azure Logic Apps**, and serves insights through a **Databricks AI/BI Dashboard** and a **Genie Agent**.

![Azure](https://img.shields.io/badge/Microsoft_Azure-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white) ![Azure Data Factory](https://img.shields.io/badge/Azure_Data_Factory-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white) ![Databricks](https://img.shields.io/badge/Databricks-FF3621?style=for-the-badge&logo=databricks&logoColor=white) ![PySpark](https://img.shields.io/badge/PySpark-E25A1C?style=for-the-badge&logo=apachespark&logoColor=white) ![SQL](https://img.shields.io/badge/SQL-4479A1?style=for-the-badge&logo=databricks&logoColor=white) ![Postgres](https://img.shields.io/badge/postgres-%23316192.svg?style=for-the-badge&logo=postgresql&logoColor=white) ![GitHub](https://img.shields.io/badge/github-%23121011.svg?style=for-the-badge&logo=github&logoColor=white)

---

## 📌 Table of Contents
- [Architecture](#-architecture)
- [Tech Stack](#-tech-stack)
- [Pipelines](#-pipelines)
- [Alerting with Logic Apps](#-alerting-with-logic-apps)
- [Analytics & Dashboard](#-analytics--dashboard)
- [Key Features](#-key-features)
- [Project Structure](#-project-structure)
- [How to Run](#-how-to-run)
- [Author](#-author)

---

## 🏗️ Architecture

```
 Neon Postgres ──┐
                 ├──► ADF (Lookup → ForEach → Switch) ──► Landing Volume (ADLS / UC Volume)
 ADLS CSV files ─┘                                              │
                                                                ▼
                                                    Databricks Notebooks
                                              Landing ➜ Bronze ➜ Silver ➜ Gold
                                                                │
                                      ┌─────────────────────────┴───────────────┐
                                      ▼                                         ▼
                           AI/BI Dashboard + Genie Agent             Logic App (Gmail alert)
```

---

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| Orchestration | Azure Data Factory (Git-integrated, `main` branch) |
| Processing | Azure Databricks, PySpark, Spark SQL |
| Storage | ADLS Gen2, Unity Catalog volumes and tables |
| Sources | Neon PostgreSQL, CSV files on ADLS |
| Alerting | Azure Logic Apps + Gmail connector |
| Analytics | Databricks AI/BI Dashboard, Genie Agent |
| Version control | GitHub |

---

## 🔄 Pipelines

The project uses a **master pipeline** that chains child pipelines, so each stage can run and be debugged on its own.

### 1. `PL_MASTER` – Orchestrator
Runs source → silver for both source types, then silver → gold, then sends the status email.

### 2. `PL_sourse_to_silver_main` – Ingest all tables
A **Lookup** reads the list of source tables from a control table, then a **ForEach** processes each table. For each table it logs *in progress*, runs the ingestion, and updates the status to *failed* if something breaks.

### 3. `PL_sourse_to_silver_inner` – Per-table ingestion
- **Lookup** reads the table details and a **Set variable** records the start time.
- **If Condition** checks the load type (full vs incremental) and reads the watermark when needed.
- **Switch** picks the source system (`neon_postgres` or `adls_csv`) and copies the data to landing.
- Databricks notebooks run **Landing → Bronze** and **Bronze → Silver**.
- A **Script** activity updates the run status, and the watermark is updated after a successful incremental load.

### 4. `PL_silver_to_gold`
Reads the list of tables, then a ForEach runs the **Silver → Gold** notebook for each one, with status logging and failure handling.

### 5. `PL_SEND_EMAIL`
Takes the pipeline `adf_run_id` as a parameter, checks the failed-run count and per-table status from the audit table, builds the email subject and HTML body, and calls the Logic App through a **Web activity**.

---

## 📧 Alerting with Logic Apps

An **HTTP-triggered Logic App** receives the run summary from ADF, converts it into an **HTML table**, and sends it as an email through the Gmail connector.

---

## 📊 Analytics & Dashboard

### MediLake Healthcare Analytics Dashboard
Built on the gold layer (`adb_medlake.gold`), covering Jan 2025 – Sep 2026. It has four pages: **Executive Overview**, **Hospital Performance**, **Lab Test Trends** and **Patient 360**.

Headline KPIs:

| Metric | Value |
|---|---|
| Total lab tests | ~8.2K |
| Patients | ~5K |
| Hospitals | 50 |
| Abnormal result rate | 22.2% |
| Critical result rate | 8.3% |
| Patients with a critical result | 13.2% |

Charts include monthly test volume by result outcome, test mix, monthly abnormal vs critical rate, patients by risk tier (normal / abnormal / critical), weekly test volume vs critical results, and hospitals by test volume.

### 🤖 Genie Agent
Business users can ask questions in plain English and get answers from the gold tables, for example *"tell me total patient count"*.

---

## ✨ Key Features

- Medallion architecture (Landing → Bronze → Silver → Gold)
- Metadata-driven ingestion using a control table, Lookup and ForEach
- Supports both full and incremental (watermark) loads
- Multi-source ingestion (PostgreSQL and ADLS CSV) using a Switch activity
- Run status tracking and failure handling per table
- Automated HTML email alerts through Logic Apps
- Git-integrated ADF development
- Self-service analytics with AI/BI Dashboard and Genie

---

## 📁 Project Structure

```
medilake/
├── adf/                  # ADF pipelines, datasets, linked services (Git-synced)
├── notebooks/            # Databricks notebooks (landing_to_bronze, bronze_to_silver, silver_to_gold)
├── sql/                  # Control table and audit table DDL
├── logic_app/            # Logic App workflow definition
└── README.md
```

---

## 🚀 How to Run

1. Create the Azure resources: Data Factory, Databricks workspace, ADLS Gen2 storage, Logic App.
2. Create the control and audit tables, and load the list of tables to ingest.
3. Import the notebooks into Databricks and create the catalog and schemas (`bronze`, `silver`, `gold`).
4. Configure the linked services (Postgres, ADLS, Databricks) in ADF.
5. Publish the Logic App and copy its HTTP POST URL into the Web activity in `PL_SEND_EMAIL`.
6. Trigger `PL_MASTER` from ADF.

---

## 👤 Author

**Ashish Kumar**
[LinkedIn](https://linkedin.com/in/ashish----kumar) · [Email](mailto:ashish11to12@gmail.com)
