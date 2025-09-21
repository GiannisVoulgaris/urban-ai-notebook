This notebook demonstrates how we used **BigQuery AI + Vertex AI** to transform 690,000+ NYC 311 service requests into actionable intelligence.  

We combined three approaches from the hackathon:

- 🕵️ **Semantic Detective**: Use embeddings + vector search to uncover hidden connections in complaints.  
- 🖼️ **Multimodal Pioneer**: Connect text queries with **real city infrastructure images**.  
- 🧠 **AI Architect**: Restructure long government response texts into clean, actionable JSON.  

> All code runs **directly in BigQuery SQL** — no external scripts. Screenshots (results, dashboards, images) are inserted after each step.

# **1. System Architecture**

All steps center around BigQuery as the data warehouse.

* Data ingestion: Raw data from NYC 311 complaints (CSV) + Images stored in GCS into BigQuery Object table.

* Remote models: Remote Models → BigQuery AI Functions  

* AI Functions: ML.GENERATE_EMBEDDING, VECTOR_SEARCH, AI.GENERATE.

* Dashboards: Structured outputs feed Looker Studio BI.

  2. Data Ingestion & Setup
We ingested 690k+ NYC 311 requests into BigQuery (citizen_complaints_final).
We also created an Object Table to reference infrastructure images stored in GCS.

Dataset: urban_ai_us (US multi-region)

BigQuery Connections:

664591046838.US.vertex_conn (Vertex AI remote models)

664591046838.US.gcs_conn (Cloud Resource for GCS)

IAM: roles/aiplatform.user granted to both connection service accounts

![1](https://raw.githubusercontent.com/GiannisVoulgaris/urban-ai-screenshots/3270f0eacf938b55c585d74235b1a9c5d96d77bf/1.png)

We use a BigQuery **Object Table** to link our GCS bucket with pothole images.  
This lets us query image files as if they were rows in a SQL table.  

```sql
-- Create an Object Table for images
CREATE OR REPLACE EXTERNAL TABLE `urban_ai_us.infrastructure_images`
WITH CONNECTION `US.gcs_conn`
OPTIONS (
  object_metadata = 'SIMPLE',
  uris = ['gs://urban-ai-final-bucket-iv/images/*.jpg']
);
```
![2](https://raw.githubusercontent.com/GiannisVoulgaris/urban-ai-screenshots/3270f0eacf938b55c585d74235b1a9c5d96d77bf/2.png)

# 3. AI Foundation: Remote Model

We create **remote models** in BigQuery that point to Vertex AI endpoints:  

- Gemini generative model
- Text embeddings  
- Multimodal embeddings (text + images)


```sql
-- LLM for AI.GENERATE
CREATE OR REPLACE MODEL `urban_ai_us.gemini_remote`
REMOTE WITH CONNECTION `projects/664591046838/locations/US/connections/vertex_conn`
OPTIONS ( endpoint = 'gemini-2.5-pro' );

-- Text embeddings
CREATE OR REPLACE MODEL `urban_ai_us.embedding_model`
REMOTE WITH CONNECTION `projects/664591046838/locations/US/connections/vertex_conn`
OPTIONS ( endpoint = 'text-multilingual-embedding-002' );

-- Multimodal embeddings (images + text)
CREATE OR REPLACE MODEL `urban_ai_us.multimodal_embedding_model`
REMOTE WITH CONNECTION `projects/664591046838/locations/US/connections/vertex_conn`
OPTIONS ( endpoint = 'multimodalembedding@001' );
```

![3](https://raw.githubusercontent.com/GiannisVoulgaris/urban-ai-screenshots/3270f0eacf938b55c585d74235b1a9c5d96d77bf/3.png)
![4](https://raw.githubusercontent.com/GiannisVoulgaris/urban-ai-screenshots/3270f0eacf938b55c585d74235b1a9c5d96d77bf/4.png)

# 4. 🧠 AI Architect — Structuring Bureaucratic Text with AI.GENERATE

The Resolution_Description field in NYC 311 data contains long, bureaucratic responses from agencies.
They are verbose and inconsistent, making them difficult to analyze.

Using BigQuery’s AI.GENERATE + Gemini 2.5 Pro, we transform them into a simple, structured schema:

* issue_type → short category name

* severity_score → integer 1–5

* summary → one-sentence summary

This step turns messy text into immediately analyzable data.

**4.1 Generate strict JSON per row (Gemini in SQL)**

```sql
CREATE OR REPLACE TABLE `urban_ai_us.structured_complaints_json` AS
SELECT
  Unique_Key,
  AI.GENERATE(
    CONCAT(
      'Return STRICT JSON only with keys: "issue_type" (short string), ',
      '"severity_score" (integer 1-5), ',
      '"summary" (one sentence). Output JSON only. Complaint: ',
      Resolution_Description
    ),
    connection_id => '664591046838.US.vertex_conn',
    endpoint => 'gemini-2.5-pro',
    output_schema => 'result STRING'
  ).result AS json_text
FROM `urban_ai_us.citizen_complaints_final`
WHERE Resolution_Description IS NOT NULL
LIMIT 500;
```

**4.2 Keep only valid JSON**

```sql
CREATE OR REPLACE TABLE `urban_ai_us.structured_complaints_json` AS
SELECT *
FROM `urban_ai_us.structured_complaints_json`
WHERE SAFE.PARSE_JSON(json_text) IS NOT NULL;
```

**4.3 Parse JSON → columns**

```sql
CREATE OR REPLACE TABLE `urban_ai_us.structured_complaints` AS
SELECT
  Unique_Key,
  JSON_VALUE(json_text, '$.issue_type') AS issue_type,
  SAFE_CAST(JSON_VALUE(json_text, '$.severity_score') AS INT64) AS severity_score,
  JSON_VALUE(json_text, '$.summary') AS summary
FROM `urban_ai_us.structured_complaints_json`;
```

![5](https://raw.githubusercontent.com/GiannisVoulgaris/urban-ai-screenshots/3270f0eacf938b55c585d74235b1a9c5d96d77bf/5.png)

![6](https://raw.githubusercontent.com/GiannisVoulgaris/urban-ai-screenshots/3270f0eacf938b55c585d74235b1a9c5d96d77bf/6.png)

**4.4 Validation: Analyzing the AI-Generated Data**

Now that we have a clean structured table, we can run analytics that were impossible before.

Top 10 Most Common Issue Types
This query performs a simple aggregation to identify the most frequent complaint categories as determined by the Gemini AI.

```sql
SELECT issue_type, COUNT(*) AS n
FROM `urban_ai_us.structured_complaints`
GROUP BY 1
ORDER BY n DESC
LIMIT 10;
```
![7](https://raw.githubusercontent.com/GiannisVoulgaris/urban-ai-screenshots/3270f0eacf938b55c585d74235b1a9c5d96d77bf/7.png)

**Severity Score Distribution**

This query shows the overall distribution of issue severity, allowing a city manager to quickly gauge the urgency of the incoming complaints.
```sql
SELECT severity_score, COUNT(*) AS n
FROM `urban_ai_us.structured_complaints`
GROUP BY 1
ORDER BY 1;
```
![8](https://raw.githubusercontent.com/GiannisVoulgaris/urban-ai-screenshots/3270f0eacf938b55c585d74235b1a9c5d96d77bf/8.png)

# 5. The Semantic Detective 🕵️‍♀️:** Finding Similar Incidents

The power of vector embeddings in BigQuery allows us to go beyond keyword search.  
Instead of just matching words, we can search *by meaning*.  

We created a new embeddings table (`urban_ai_us.resolution_embeddings`) from the **Resolution_Description** field — the long, bureaucratic responses from agencies.  
Using `ML.GENERATE_EMBEDDING` + `VECTOR_SEARCH`, we can now run semantic queries like:

- "Police issued a summons"
- "Health department found violations"
- "DEP cleaned the catch basin"

This makes it possible for city managers to instantly retrieve and group together related incidents, even if the text differs in wording.
```sql
-- Step 1: Create resolution embeddings (once)
CREATE OR REPLACE TABLE `urban_ai_us.resolution_embeddings` AS
SELECT
  Unique_Key,
  ml_generate_embedding_result AS embedding
FROM ML.GENERATE_EMBEDDING(
  MODEL `urban_ai_us.embedding_model`,
  (
    SELECT
      Unique_Key,
      Resolution_Description AS content
    FROM `urban_ai_us.citizen_complaints_final`
    WHERE Resolution_Description IS NOT NULL
      AND UPPER(Resolution_Description) <> 'N/A'
      AND LENGTH(Resolution_Description) > 50
    LIMIT 20000
  ),
  STRUCT(TRUE AS flatten_json_output, 'RETRIEVAL_DOCUMENT' AS task_type)
);
```
![9](https://raw.githubusercontent.com/GiannisVoulgaris/urban-ai-screenshots/3270f0eacf938b55c585d74235b1a9c5d96d77bf/9.png)
```sql
-- Step 3a: Search for "police issued a summons"
WITH q AS (
  SELECT ml_generate_embedding_result AS qvec
  FROM ML.GENERATE_EMBEDDING(
    MODEL `urban_ai_us.embedding_model`,
    (SELECT "police issued a summons" AS content),
    STRUCT(TRUE AS flatten_json_output, 'RETRIEVAL_QUERY' AS task_type)
  )
)
SELECT
  vs.base.Unique_Key,
  c.Resolution_Description,
  vs.distance
FROM VECTOR_SEARCH(
  TABLE `urban_ai_us.resolution_embeddings`,
  'embedding',
  (SELECT qvec FROM q),
  top_k => 10,
  distance_type => 'COSINE'
) AS vs
JOIN `urban_ai_us.citizen_complaints_final` c
  ON c.Unique_Key = vs.base.Unique_Key
ORDER BY vs.distance ASC;
```
![10](https://raw.githubusercontent.com/GiannisVoulgaris/urban-ai-screenshots/3270f0eacf938b55c585d74235b1a9c5d96d77bf/10.png)

-- Step 3b: Search for "health department found violations"
```sql
WITH q AS (
  SELECT ml_generate_embedding_result AS qvec
  FROM ML.GENERATE_EMBEDDING(
    MODEL `urban_ai_us.embedding_model`,
    (SELECT "health department found violations and scheduled follow-up inspections" AS content),
    STRUCT(TRUE AS flatten_json_output, 'RETRIEVAL_QUERY' AS task_type)
  )
)
SELECT
  vs.base.Unique_Key,
  c.Resolution_Description,
  vs.distance
FROM VECTOR_SEARCH(
  TABLE `urban_ai_us.resolution_embeddings`,
  'embedding',
  (SELECT qvec FROM q),
  top_k => 10,
  distance_type => 'COSINE'
) AS vs
JOIN `urban_ai_us.citizen_complaints_final` c
  ON c.Unique_Key = vs.base.Unique_Key
ORDER BY vs.distance ASC;
```
![11](https://raw.githubusercontent.com/GiannisVoulgaris/urban-ai-screenshots/3270f0eacf938b55c585d74235b1a9c5d96d77bf/11.png)
```sql
WITH q AS (
  SELECT ml_generate_embedding_result AS qvec
  FROM ML.GENERATE_EMBEDDING(
    MODEL `urban_ai_us.embedding_model`,
    (SELECT "DEP cleaned the catch basin" AS content),
    STRUCT(TRUE AS flatten_json_output, 'RETRIEVAL_QUERY' AS task_type)
  )
)
SELECT
  vs.base.Unique_Key,
  c.Resolution_Description,
  vs.distance
FROM VECTOR_SEARCH(
  TABLE `urban_ai_us.resolution_embeddings`,
  'embedding',
  (SELECT qvec FROM q),
  top_k => 10,
  distance_type => 'COSINE'
) AS vs
JOIN `urban_ai_us.citizen_complaints_final` c
  ON c.Unique_Key = vs.base.Unique_Key
ORDER BY vs.distance ASC;
```
![12](https://raw.githubusercontent.com/GiannisVoulgaris/urban-ai-screenshots/3270f0eacf938b55c585d74235b1a9c5d96d77bf/12.png)


# 6. The Multimodal Pioneer 🖼️: Text → Image Search

City managers often face complaints that are vague or hard to verify. For example: “a dangerous pothole in the road.”

With BigQuery’s multimodal embeddings, we can bridge text and images in the same semantic space. This allows a simple text query to retrieve the most relevant photographic evidence.

Demo:

* Generate a vector embedding for the phrase “a dangerous pothole in the road.”

* Search across our pre-computed image embeddings with VECTOR_SEARCH.

* Instantly return the top 10 most similar photos.

-- Step 1: Generate the vector for a text query using the multimodal model
```sql
WITH search_query AS (
  SELECT ml_generate_embedding_result AS search_vector
  FROM ML.GENERATE_EMBEDDING(
    MODEL `urban_ai_us.multimodal_embedding_model`, 
    (SELECT "a dangerous pothole in the road" AS content),
    STRUCT(TRUE AS flatten_json_output)
  )
)
-- Step 2: Use that vector to find the top 10 most visually similar images

SELECT
  search_results.base.uri,
  search_results.distance
FROM VECTOR_SEARCH(
  TABLE `urban_ai_us.image_embeddings`,
  'embedding',
  (SELECT search_vector FROM search_query),
  top_k => 10,
  distance_type => 'COSINE'
) AS search_results
ORDER BY search_results.distance;
```
![Pothole Examples](https://raw.githubusercontent.com/GiannisVoulgaris/urban-ai-screenshots/3270f0eacf938b55c585d74235b1a9c5d96d77bf/potholes.png)


# 7. Actionable Insights: Anomalies & Hotspots 🔎

Beyond semantic and multimodal search, Urban AI also surfaces patterns in the data.

Using standard BigQuery SQL (no extra infra), we can:

* Detect anomalies — flag sudden spikes in complaints compared to the historical rolling average.

* Map hotspots — geospatial aggregation pinpoints problem areas in the city.

These insights are designed to feed directly into BI dashboards (e.g., Looker Studio), giving decision-makers a live pulse on the city.


-- Create a view that compares daily complaint counts against a 30-day rolling average
```sql
CREATE OR REPLACE VIEW `urban_ai_us.v_anomalies` AS
WITH daily_counts AS (
  SELECT
    Complaint_Type,
    DATE(Created_Date) AS day,
    COUNT(*) AS num_complaints
  FROM `urban_ai_us.citizen_complaints_final`
  GROUP BY Complaint_Type, day
)
SELECT
  Complaint_Type,
  day,
  num_complaints,
  AVG(num_complaints) OVER (
    PARTITION BY Complaint_Type 
    ORDER BY day 
    ROWS BETWEEN 30 PRECEDING AND 1 PRECEDING
  ) AS rolling_avg_30_day
FROM daily_counts;
```
```sql
-- Create a view that aggregates complaints into geospatial hotspots

CREATE OR REPLACE VIEW `urban_ai_us.v_hotspots` AS
SELECT
  ST_GEOGPOINT(longitude, latitude) AS location,
  Complaint_Type,
  COUNT(*) AS num_complaints
FROM `urban_ai_us.citizen_complaints_final`
WHERE latitude IS NOT NULL AND longitude IS NOT NULL
GROUP BY 
  latitude, 
  longitude, 
  Complaint_Type;
```

  # 8. Locker Studio

[Locker Studio Link](https://lookerstudio.google.com/reporting/85845d0f-0d8e-4a79-a38a-1e333c85a219)


![Step 14 – Geographic Hotspots Map](https://raw.githubusercontent.com/GiannisVoulgaris/urban-ai-screenshots/3270f0eacf938b55c585d74235b1a9c5d96d77bf/14.png)



--- 
## 📝 Team Survey: BigQuery AI - Building the Future of Data
### 1. BigQuery AI Experience

- Team Member: Ioannis Voulgaris — 0 months prior experience (all learning happened during hackathon)

### 2. Google Cloud Experience

- Team Member: Ioannis Voulgaris — 0 months prior experience (first-time user of BigQuery, Vertex AI, IAM, Object Tables, Looker Studio, etc.)

### 3. Feedback

This was my very first experience with BigQuery AI and Google Cloud. I faced several real-world challenges along the way:

Parsing large CSVs

Ingesting the 690k+ rows of NYC 311 service requests was harder than expected.

The standard BigQuery UI loader repeatedly failed.

Solution: I pivoted to Vertex AI Notebooks with pandas to load the dataset reliably.

Permissions & IAM complexity

BigQuery AI functions (AI.GENERATE_TEXT, ML.GENERATE_EMBEDDING) initially failed.

Root cause: connection service accounts lacked roles/aiplatform.user.

Solution: Explicitly granted IAM roles to hidden service accounts generated by BigQuery connections.

AI Functions configuration

Documentation around remote models, connections, and AI.GENERATE syntax was often unclear.

I hit errors like “Model not found” or “Table-valued function not found”.

Once I used the right combo (connection_id, endpoint, output_schema), everything worked smoothly.

Time & scale issues

Some queries (e.g., generating structured JSON for 500 rows) ran for 30+ minutes.

Solution: batched smaller queries; but clearer guidance or defaults would improve developer UX.

Despite these hurdles, I came away very impressed:

✅ BigQuery AI power: Running embeddings, vector search, and multimodal text→image queries directly in SQL is a game-changer.

✅ Real-world potential: I demonstrated that messy 311 complaints + GCS images can become structured, AI-powered insights all inside BigQuery.

💡 Improvement idea: A guided “AI setup wizard” (automatic IAM bindings + sample queries) would make onboarding easier for first-time users.

Overall: The learning curve was steep, but this hackathon showed me the real strength of BigQuery AI. I now feel confident using it for real-world projects.
