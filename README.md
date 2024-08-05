## Automating Insights and Dashboard Report-Level Narratives using Vertex AI Gemini 1.5-Flash, Looker and Google BigQuery BQML

This document outlines a solution for automating data insights and generating report-level narratives for Looker dashboards using Google Cloud's Vertex AI Gemini 1.5-Flash large language model (LLM) and BigQuery.

### Overview

The traditional process of producing monthly report packs involves five key steps:

1. **Data Collection:** Gathering data for various KPIs.
2. **Data Preparation:** Cleaning, validating, and structuring the data for analysis.
3. **Report Assembly & Publishing:** Creating and distributing initial reports, usually to an analyst.
4. **Analyst Review & Interpretation:** Comparing current data against historical trends and external factors.
5. **Narrative Generation:** Translating the analysis into digestible summaries for senior management.

While data pipeline tools and BI platforms have streamlined the first three steps, steps 4 and 5 often remain manual bottlenecks. This solution leverages the power of LLMs to automate insight generation and narrative creation, accelerating the reporting process and enhancing decision-making.

### Solution Architecture

This solution utilizes various Google Cloud and third-party products and services:

- **Looker:** The primary BI tool used to build and visualize the final reports and dashboards.
- **Google BigQuery:** The data warehouse storing the historical KPI data used for analysis and narrative generation.
- **Dataform (or similar ETL tool):** Used to transform and aggregate raw data into a monthly KPI summary table in BigQuery.
- **Vertex AI:** Provides access to the Gemini 1.5-Flash LLM, enabling advanced text generation and analysis.

### Detailed Implementation Steps

#### 1. Data Collection and Preparation

1. **Create Looker Looks:** Develop Looker looks (reports) to aggregate and visualize the relevant KPIs, for example revenue and sales KPIs, by store, period and high-level sales category
2. **Extract SQL Queries:** Utilize the "View SQL" feature in Looker to obtain the underlying SQL queries for the defined looks.
3. **Data Transformation with Dataform:**
- Utilize the extracted SQL queries as the basis for Dataform configurations.
- Create a Dataform pipeline to transform and aggregate the raw data into a monthly (or weekly) KPI summary table in BigQuery.

#### 2. Data Formatting

Create a BigQuery table named `monthly_performance_fact` (or similar) to store the aggregated KPI data:

```sql
CREATE TABLE `your_project.your_dataset.monthly_performance_fact`
(
date_spine_dim_date_month STRING,
total_new_data_centralization_sales_leads INT64,
total_new_marketing_analytics_sales_leads INT64,
/* ... other relevant KPI columns ... */
pct_cost_of_delivery FLOAT64,
pct_cost_of_overheads FLOAT64
);
```

Populate this table using the Dataform pipeline, ensuring the `date_spine_dim_date_month` column is formatted as `YYYY-MM`.

#### 3. Formatting the KPI History Table as a JSON String

Transform the tabular KPI history data into a JSON string format suitable for the LLM prompt:

```sql
WITH
monthly_performance_data AS (
SELECT
CONCAT('[',STRING_AGG(TO_JSON_STRING(STRUCT(date_spine_dim_date_month,
total_new_data_centralization_sales_leads,
total_new_marketing_analytics_sales_leads,
/* ... other relevant KPI columns ... */
pct_cost_of_delivery,
pct_cost_of_overheads) )
ORDER BY
date_spine_dim_date_month DESC),']') AS monthly_metrics
FROM
`your_project.your_dataset.monthly_performance_fact`
WHERE
CONCAT(date_spine_dim_date_month,'-01') != CAST(DATE_TRUNC(CURRENT_DATE,MONTH) AS STRING)
)

SELECT * FROM monthly_performance_data
```

This query generates a single-row table with a `monthly_metrics` column containing a JSON string representing the entire KPI history, with the most recent month's data at the beginning.

#### 4. Creating the Vertex AI Gemini 1.5 Flash LLM Model in Google BigQuery

1. **Establish Cloud Resource Connection:**
- Navigate to the "Connections" section in your BigQuery project in the Google Cloud Console.
- Create a new connection to Vertex AI and grant the necessary IAM permissions for BigQuery to access the Vertex AI API.
2. **Register the LLM Model:**
- In BigQuery, execute the following SQL statement to register the Gemini 1.5 Flash model:

```sql
CREATE OR REPLACE MODEL `your_project.your_dataset.gemini_1_5_flash`
REMOTE WITH CONNECTION `your_project.your_location.your_vertex_ai_connection`
OPTIONS (endpoint = 'gemini-1.5-flash')
```

- Replace placeholders with your specific project ID, location, and connection name.

#### 5. Sending a Prompt and the Metrics JSON String to the LLM

Use the `ML.GENERATE_TEXT` function in BigQuery to send prompts and the formatted JSON data to the registered LLM:

```sql
WITH
monthly_performance_data AS (
/* ... (previous query to generate monthly_metrics JSON) ... */
),

overall_summary AS (
SELECT
DATE_TRUNC(current_date,MONTH) AS analysis_month,
ml_generate_text_llm_result AS overall_summary
FROM
ML.GENERATE_TEXT( MODEL `your_project.your_dataset.gemini_1_5_flash`,
(
SELECT
CONCAT('This data is for [Your Company Name], a [Company Description]. Please give me a summary of performance in a multi-paragraph format suitable for inclusion as narrative in a dashboard and with no header or greeting. Analyze the latest month in this dataset with the CEO and SMT as the audience, comparing against the previous month and current quarter-to-date to the previous quarter-to-date and current month to the same month last year, highlighting the most significant changes and trends in data and analyzing how the impact of changes in one metric are affecting others either within that month or over time.: ', mm.monthly_metrics) AS prompt
FROM
monthly_performance_data mm
),
STRUCT( 0.2 AS temperature, 1024 AS max_output_tokens, TRUE AS flatten_json_output)
)
)

SELECT * FROM overall_summary
```

This query generates a narrative summarizing the overall performance based on the provided KPI data and the prompt instructions. Adjust the prompt as needed to tailor the analysis and narrative style.

#### 6. Visualizing the Narratives in Looker

1. **Create a Looker LookML View:** Define a new LookML view based on the BigQuery table containing the generated narratives.
2. **Add Table (Report) Visualization:** In your Looker dashboard, incorporate a "Table (Report)" visualization using the new LookML view as its data source. This table will display the generated narratives alongside your other KPI visualizations.

### Google Cloud Roles and Permissions

To implement this solution, Rittman Analytics developers require the following Google Cloud roles and permissions and APIs enabled:

- **BigQuery:**
- `bigquery.admin`: Required for creating datasets, tables, and models.
- `bigquery.user`: Required for querying data and running models.
- `bigquery.jobUser`: Required for creating and managing BigQuery ML jobs.

- **Vertex AI:**
- `aiplatform.user`: Required for interacting with Vertex AI models.
- `iam.serviceAccountUser`: Required for creating and managing service accounts for accessing Vertex AI.

- **Cloud Resource Manager:**
- `resourcemanager.projectViewer`: Required for listing and viewing project details.

### Google Cloud APIs

- **BigQuery API**
- **Vertex AI API**

### Conclusion

This solution demonstrates how to leverage Vertex AI's Gemini 1.5-Flash LLM and BigQuery's ML capabilities to automate the generation of insightful narratives for Looker dashboards. By automating this previously manual and time-consuming process, businesses can improve reporting efficiency, provide richer context for decision-making, and gain a deeper understanding of their data.
