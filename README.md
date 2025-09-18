# Comprehensive Step-by-Step Guide: Orchestrating Data Pipelines with Airflow, BigQuery, and dbt

This guide explains how to build a data pipeline leveraging Apache Airflow, Google BigQuery, and dbt to automate healthcare-related data workflows. Each section provides clear, actionable instructions. This project was built and documented by Sujeeth

---

## Data Pipeline Overview
- **Overview**: The diagram depicts the pipeline workflow orchestrated through Airflow, BigQuery, and dbt. It shows each stage from generating raw data with a Python script (uploaded to Google Cloud Storage), to creating BigQuery external tables, performing initial data quality checks, and running dbt models with tests.
- **Diagram**: ![Data Pipeline Orchestration Diagram](images/airflow_orchastration.png)

---

## Table of Contents
- [Prerequisites](#prerequisites)
- [Step 1: Set Up Your Development Environment](#step-1-set-up-your-development-environment)
- [Step 2: Set Up Google Cloud Credentials](#step-2-set-up-google-cloud-credentials)
- [Step 3: Create the Project Structure](#step-3-create-the-project-structure)
- [Step 4: Modify the Dockerfile for Dependencies](#step-4-modify-the-dockerfile-for-dependencies)
- [Step 5: Create the Airflow DAG](#step-5-create-the-airflow-dag)
- [Step 6: Create External Tables in BigQuery](#step-6-create-external-tables-in-bigquery)
- [Step 7: Set Up dbt and Run Tests](#step-7-set-up-dbt-and-run-tests)
- [Step 8: Run dbt Transformations](#step-8-run-dbt-transformations)
- [Step 9: Switch Between Dev and Prod Environments](#step-9-switch-between-dev-and-prod-environments)
- [Step 10: Test and Deploy](#step-10-test-and-deploy) 
- [Result](#result)
- [Additional Tips](#additional-tips)
- [Resources](#resources)

---

## Prerequisites
- **Purpose**: Confirm you have all tools and accounts ready before beginning.
- **Requirements**:
  1. **Google Cloud Platform (GCP) Account**: Create a project (e.g., `healthcare-data-project-442109`) with billing enabled and activate both GCS and BigQuery.
     - [Google Cloud Documentation](https://cloud.google.com/docs)
  2. **Python 3.8 or higher**: Verify using `python --version`.
  3. **Astro CLI**: Used to run Airflow locally.
     - [Astro CLI Installation Guide](https://docs.astronomer.io/astro/cli/install-cli)
  4. **dbt Core**: For running transformations in BigQuery.
     - [dbt Core Installation](https://docs.getdbt.com/docs/core/installation)
  5. **Foundational Knowledge**: Basic understanding of Airflow, dbt, and BigQuery.
     - [Airflow Docs](https://airflow.apache.org/docs/) | [dbt Docs](https://docs.getdbt.com/) | [BigQuery Docs](https://cloud.google.com/bigquery/docs)

---

## Step 1: Set Up Your Development Environment
- **Purpose**: Configure your local environment for running Airflow and building the pipeline.
- **Instructions**:
  1. **Install Astro CLI**:
     - Follow the Astronomer installation instructions and confirm with `astro --version`.
  2. **Initialize the Astro Project**:
     - Run: `astro dev init`
     - This sets up the initial structure (`dags`, `include`, etc.).
  3. **Launch the Development Server**:
     - `cd <your-project-name>`
     - Start Airflow: `astro dev start`
     - Access Airflow at `http://localhost:8080`.

---

## Step 2: Set Up Google Cloud Credentials
- **Purpose**: Allow Airflow to authenticate and access BigQuery and GCS.
- **Instructions**:
  1. **Generate a Service Account**:
     - In the GCP Console, navigate to **IAM & Admin > Service Accounts**, assign roles (`BigQuery Admin`, `Storage Admin`), and download `service_account.json`.
  2. **Link GCP in Airflow**:
     - Place `service_account.json` in `/usr/local/airflow/include/gcp/`.
     - Add to `.env`:
       ```
       AIRFLOW__CORE__TEST_CONNECTION=Enabled
       ``` 
     - In Airflow UI (**Admin > Connections**):
       - **Connection ID**: `gcp`
       - **Type**: `Google Cloud`
       - **Project ID**: `healthcare-data-project-442109`
       - **Keyfile**: `/usr/local/airflow/include/gcp/service_account.json`
     - Click **Test** to confirm.

---

## Step 3: Create the Project Structure
- **Purpose**: Arrange files for smooth project management.
- **Instructions**:
  1. **Organize the `include` Folder**:
     - Create subfolders:
       - `raw_data_generation` for scripts.
       - `gcp` for credentials.
     - Place the `dbt` folder at the **project root**, not inside `include`.
  2. **Add Data Generation Script**:
     - Save `healthcare_data.py` in `raw_data_generation` to create sample files and upload them to GCS (e.g., `gs://healthcare-data-bucket-emeka/dev/`).
     - [View Script](./include/raw_data_generation/healthcare_data.py)

---

## Step 4: Modify the Dockerfile for Dependencies
- **Purpose**: Install all required libraries within an isolated environment.
- **Instructions**:
  1. **Edit `Dockerfile`**:
     ```
     FROM quay.io/astronomer/astro-runtime:12.6.0

     ENV VENV_PATH="/usr/local/airflow/dbt_venv"
     ENV PATH="$VENV_PATH/bin:$PATH"

     RUN python -m venv $VENV_PATH && \
         source $VENV_PATH/bin/activate && \
         pip install --upgrade pip setuptools && \
         pip install --no-cache-dir dbt-bigquery==1.5.3 pandas Faker pyarrow numpy && \
         deactivate

     RUN echo "source $VENV_PATH/bin/activate" > /usr/local/airflow/dbt_env.sh
     RUN chmod +x /usr/local/airflow/dbt_env.sh
     ```
  2. **Restart the Environment**:
     - Run: `astro dev restart`

---

## Step 5: Create the Airflow DAG
- **Purpose**: Define your workflow using an Airflow DAG.
- **Instructions**:
  1. **Make a DAG File**:
     - Create `healthcare_pipeline_dag.py` inside `dags`.
  2. **Import Needed Libraries**:
     ```python
     from airflow.decorators import dag
     from pendulum import datetime
     from airflow.operators.bash import BashOperator
     from airflow.providers.google.cloud.operators.bigquery import BigQueryInsertJobOperator
     from cosmos.airflow.task_group import DbtTaskGroup
     from cosmos.constants import LoadMode
     from cosmos.config import RenderConfig, ProfileConfig, ProjectConfig
     from pathlib import Path
     ```
  3. **Declare the DAG**:
     ```python
     @dag(
         schedule=None,
         start_date=datetime(2024, 1, 1),
         catchup=False,
         tags=["healthcare"],
         doc_md="Orchestrates healthcare data pipeline with BigQuery and dbt"
     )
     def healthcare_pipeline():
         pass
     ```
  4. **Add Task 1: Data Generation**:
     ```python
     PATH_TO_DATA_SCRIPT = "/usr/local/airflow/include/raw_data_generation/healthcare_data.py"
     generate_data = BashOperator(
         task_id="generate_data",
         bash_command=f"python {PATH_TO_DATA_SCRIPT}"
     )
     ```

---

## Step 6: Create External Tables in BigQuery
- **Purpose**: Connect raw files in GCS directly to BigQuery.
- **Instructions**:
  1. **Write SQL File** (`create_external_tables.sql`):
     ```sql
     CREATE OR REPLACE EXTERNAL TABLE ...
     ```
     - External tables let you query GCS data without moving it into BigQuery.
  2. **Add to DAG**:
     ```python
     PATH_TO_SQL_SCRIPT = "/usr/local/airflow/include/raw_data_generation/create_external_tables.sql"
     with open(PATH_TO_SQL_SCRIPT, "r") as f:
         CREATE_EXTERNAL_TABLES_SQL = f.read()
     ...
     generate_data >> create_external_tables
     ```

---

## Step 7: Set Up dbt and Run Tests
- **Purpose**: Configure dbt (now located in the root) and test your data sources.
- **Instructions**:
  1. **Create a dbt Project** inside `dbt/healthcare_dbt_bigquery_data_pipeline`.
  2. **Add Config Files** (`dbt_project.yml`, `profiles.yml`, `cosmos_config.py`).
  3. **Create a Test Task** in the DAG:
     ```python
     dbt_test_raw = BashOperator(
         task_id="dbt_test_raw",
         bash_command="source /usr/local/airflow/dbt_venv/bin/activate && dbt test --select source:*",
         cwd="/usr/local/airflow/dbt/healthcare_dbt_bigquery_data_pipeline"
     )
     ```

---

## Step 8: Run dbt Transformations
- **Purpose**: Execute dbt models to clean and reshape your data.
- **Instructions**:
  1. **Add the Transformation Task**:
     ```python
     transform = DbtTaskGroup(...)
     ```
  2. **Confirm Dependencies** in `requirements.txt`:
     ```
     apache-airflow-providers-google
     astronomer-cosmos==1.8.2
     ```

---

## Step 9: Switch Between Dev and Prod Environments
- **Purpose**: Easily toggle between dev and prod.
- **Instructions**:
  1. **In `cosmos_config.py`**, change `target_name` to `prod`.
  2. **In the DAG**, dynamically replace `dev` with `prod` in SQL paths.

---

## Step 10: Test and Deploy
- **Purpose**: Validate locally and push to production.
- **Instructions**:
  1. **Local Test**:
     ```bash
     astro dev bash
     source /usr/local/airflow/dbt_venv/bin/activate
     cd /usr/local/airflow/dbt/healthcare_dbt_bigquery_data_pipeline
     dbt test --select source:*
     dbt run --select path:models
     ```
  2. **Trigger the DAG** from the Airflow UI.
  3. **Deploy** using `astro deploy` after switching to `prod`.

---

## Result
- **Purpose**: Summarizes what the pipeline delivers.
- **Outcome**: This setup generates raw data, builds external tables in BigQuery, runs dbt transformations, and produces tested, curated datasets (e.g., `health_anomalies`, `patient_demographics`). Raw data is stored in GCS buckets (`healthcare-data-bucket-emeka`).
- **Visuals**:
  - DAG View: ![healthcare_pipeline DAG](images/Airflow.png)
  - BigQuery Table: ![health_anomalies](images/BigQuery.png)
  - GCS Bucket: ![healthcare-data-bucket-emeka](images/GCS.png)

---

## Additional Tips
- Add `on_failure_callback` to log or send alerts on errors.
- Monitor pipelines via Airflow UI or set up Slack alerts.
- Store credentials safely using Airflow secrets or environment variables.

---

## Resources
- [Airflow Tutorials](https://airflow.apache.org/docs/apache-airflow/stable/tutorial.html)
- [dbt BigQuery Setup](https://docs.getdbt.com/reference/warehouse-setups/bigquery-setup)
- [BigQuery External Tables](https://cloud.google.com/bigquery/docs/external-data-sources)
- [Astronomer Cosmos Docs](https://github.com/astronomer/astronomer-cosmos)

---

# Astro Project Structure (Post `astro dev init`)

This README explains what’s created by the Astronomer CLI after running `astro dev init` and how to use it to run Airflow locally.

## Project Contents
- **dags**: Contains DAG Python scripts. Comes with an `example_astronauts` DAG showing a simple ETL that prints astronaut names from the Open Notify API.
- **Dockerfile**: Specifies the Astro Runtime image and any runtime overrides.
- **include**: Place any extra project files here (empty initially).
- **packages.txt**: For OS-level package installations (empty initially).
- **requirements.txt**: For Python dependencies (empty initially).
- **plugins**: Add custom/community plugins (empty initially).
- **airflow_settings.yaml**: Store local-only Airflow settings (connections, variables, pools).

## Running Locally
1. Start Airflow: `astro dev start`
2. Confirm containers: `docker ps`
3. Visit `http://localhost:8080` and log in with `admin` / `admin`.

## Deploying to Astronomer
Push your code to Astronomer by following the [deployment guide](https://www.astronomer.io/docs/astro/deploy-code/).

## Contact
The Astronomer CLI is maintained by the Astronomer team. Reach out to their support to report issues or request changes.
