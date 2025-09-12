# 🚇 London Travel Medallion Project  

## 📌 Project Overview  
This project demonstrates how to build a **modern data engineering pipeline** using **Azure services** and the **Medallion Architecture (Bronze → Silver → Gold)**.  

It focuses on processing three key datasets — **Journey**, **Passengers**, and **Stations** — to produce business-ready insights, enabling reporting and visualization in **Power BI**.  

The solution leverages **Azure Data Factory, Azure Databricks, Azure Data Lake Storage (ADLS Gen2), Azure Key Vault**, and **Power BI**.  

---

## 🏗️ Solution Architecture  

The project uses the Medallion architecture pattern:  

1. **Bronze Layer** – Stores raw ingested data with minimal transformation.  
2. **Silver Layer** – Cleans and standardizes datasets.  
3. **Gold Layer** – Business-ready tables and KPIs for dashboards.  

### Flow  
**Source Data → ADF → ADLS (Bronze → Silver → Gold via Databricks) → Power BI Dashboard**  

![Solution Architecture](images/architecture.png)  


---

## 📂 Datasets  

- **Journey** – Travel history logs (origin, destination, start/end time, ticket type).  
- **Passengers** – Customer demographics, profiles, and ticket purchases.  
- **Stations** – Station reference data (station IDs, locations, and zones).  

---

## ⚙️ Azure Services Used  

- **Azure Data Factory (ADF):** Orchestration and pipeline management.  
- **Azure Databricks:** PySpark-based transformations across Bronze, Silver, and Gold.  
- **Azure Data Lake Storage (ADLS Gen2):** Centralized storage for raw and processed data.  
- **Azure Key Vault:** Secure storage for access keys, secrets, and tokens.  
- **Power BI:** Visualization and analytics dashboards.  

---

## 🔄 PySpark Transformations  

### 1. Journey Dataset  

**Bronze → Silver (Cleaning & Standardizing):**  
```python
from pyspark.sql import functions as F

journey_bronze = spark.read.parquet("abfss://bronze@<storage_account>.dfs.core.windows.net/journey/")

journey_silver = (
    journey_bronze
    .withColumn("journey_date", F.to_date("timestamp"))
    .withColumn("duration_mins", (F.col("end_time").cast("long") - F.col("start_time").cast("long"))/60)
    .filter(F.col("origin_station").isNotNull() & F.col("destination_station").isNotNull())
)

journey_silver.write.mode("overwrite").parquet("abfss://silver@<storage_account>.dfs.core.windows.net/journey/")
```

**Silver → Gold (Aggregations for Insights):**  
```python
journey_gold = (
    journey_silver
    .groupBy("journey_date", "origin_station", "destination_station")
    .agg(
        F.count("*").alias("journey_count"),
        F.avg("duration_mins").alias("avg_duration")
    )
)

journey_gold.write.mode("overwrite").parquet("abfss://gold@<storage_account>.dfs.core.windows.net/journey/")
```

---

### 2. Passengers Dataset  

**Bronze → Silver:**  
```python
passengers_bronze = spark.read.parquet("abfss://bronze@<storage_account>.dfs.core.windows.net/passengers/")

passengers_silver = (
    passengers_bronze
    .dropDuplicates(["passenger_id"])
    .withColumn("age", F.col("age").cast("int"))
    .withColumn("signup_date", F.to_date("signup_date"))
    .filter(F.col("passenger_id").isNotNull())
)

passengers_silver.write.mode("overwrite").parquet("abfss://silver@<storage_account>.dfs.core.windows.net/passengers/")
```

**Silver → Gold:**  
```python
passengers_gold = (
    passengers_silver
    .groupBy("gender", "signup_date")
    .agg(
        F.countDistinct("passenger_id").alias("unique_passengers"),
        F.avg("age").alias("avg_age")
    )
)

passengers_gold.write.mode("overwrite").parquet("abfss://gold@<storage_account>.dfs.core.windows.net/passengers/")
```

---

### 3. Stations Dataset  

**Bronze → Silver:**  
```python
stations_bronze = spark.read.parquet("abfss://bronze@<storage_account>.dfs.core.windows.net/stations/")

stations_silver = (
    stations_bronze
    .dropDuplicates(["station_id"])
    .withColumn("zone", F.col("zone").cast("int"))
    .filter(F.col("station_name").isNotNull())
)

stations_silver.write.mode("overwrite").parquet("abfss://silver@<storage_account>.dfs.core.windows.net/stations/")
```

**Silver → Gold:**  
```python
stations_gold = (
    stations_silver
    .select("station_id", "station_name", "zone", "location")
)

stations_gold.write.mode("overwrite").parquet("abfss://gold@<storage_account>.dfs.core.windows.net/stations/")
```

---

## 📜 ADF Pipeline  

### Pipeline Flow  
1. **Copy datasets (Journey, Passengers, Stations) into Bronze**.  
2. **Trigger Databricks notebooks** for Silver transformations.  
3. **Run Gold-level transformations** in Databricks.  
4. **(Optional)** Trigger Power BI dataset refresh.  

### Simplified Pipeline Diagram  
```
[Copy to Bronze] → [Databricks Silver] → [Databricks Gold] → [Power BI Dashboard]
```

### Sample Pipeline JSON (Journey)  
```json
{
  "name": "LondonMedallionPipeline",
  "properties": {
    "activities": [
      {
        "name": "Copy Journey to Bronze",
        "type": "Copy",
        "typeProperties": {
          "source": { "type": "DelimitedTextSource" },
          "sink": { "type": "ParquetSink" }
        },
        "inputs": [{ "referenceName": "JourneyRawDataset", "type": "DatasetReference" }],
        "outputs": [{ "referenceName": "JourneyBronzeDataset", "type": "DatasetReference" }]
      },
      {
        "name": "Run Databricks Silver Transformations",
        "type": "DatabricksNotebook",
        "dependsOn": [{ "activity": "Copy Journey to Bronze", "dependencyConditions": ["Succeeded"] }],
        "typeProperties": {
          "notebookPath": "/Shared/medallion/silver_transformations",
          "baseParameters": { "input": "bronze/journey", "output": "silver/journey" }
        },
        "linkedServiceName": { "referenceName": "AzureDatabricksService", "type": "LinkedServiceReference" }
      },
      {
        "name": "Run Databricks Gold Transformations",
        "type": "DatabricksNotebook",
        "dependsOn": [{ "activity": "Run Databricks Silver Transformations", "dependencyConditions": ["Succeeded"] }],
        "typeProperties": {
          "notebookPath": "/Shared/medallion/gold_transformations",
          "baseParameters": { "input": "silver/journey", "output": "gold/journey" }
        },
        "linkedServiceName": { "referenceName": "AzureDatabricksService", "type": "LinkedServiceReference" }
      }
    ]
  }
}
```

---

## 📊 Final Outputs  

- **Bronze:** Raw data (`Journey`, `Passengers`, `Stations`)  
- **Silver:** Cleaned & standardized datasets (deduplicated, validated, enriched)  
- **Gold:** Business-ready KPIs and aggregated datasets for reporting  

**Power BI Dashboard Insights:**  
- Passenger journey trends by station and time  
- Station performance KPIs  
- Ticket sales & revenue analysis  
- ROI and efficiency metrics  

---

## 🚀 Future Enhancements  
- Add **CI/CD integration** with GitHub and ADF for automated deployments.  
- Enable **real-time ingestion** via Event Hubs or Kafka.  
- Extend with **predictive analytics** in Databricks ML.  

---

✅ This repo demonstrates how to design and deploy a **scalable data pipeline** in Azure using the **Medallion architecture** for the **London Travel Medallion project**.  
