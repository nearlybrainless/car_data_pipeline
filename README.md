# 🚗 Car Data Engineering Project

## 📋 Project Overview
This project implements an end-to-end data engineering solution for car sales data using Azure cloud services. The pipeline utilizes a robust medallion architecture (Bronze, Silver, Gold) with dimensional modeling, enabling both initial and incremental data loads through a parameterized approach. The solution leverages Azure Data Factory for orchestration, Azure Databricks for transformation, and Unity Catalog for metadata management.

## 🏗️ Architecture
![Architecture Diagram](https://github.com/nearlybrainless/car_data_pipeline/blob/main/Architecture%20Diagram.jpg)

## 🧩 Components
- **Data Source**: 📁 Git repository containing car sales data
- **Data Processing**: ⚙️ Azure Data Factory (ADF) for orchestration
- **Data Storage**: 
  - 💾 Azure SQL Database (Source preparation)
  - 🌊 Azure Data Lake Storage Gen2 (Bronze layer)
  - ✨ Databricks Unity Catalog (Silver and Gold layers)
- **Processing Engine**: 🔥 Azure Databricks notebooks
- **Data Management**: 📊 Delta Lake for ACID transactions

## 🔄 Pipeline Flow

![Pipeline](https://github.com/nearlybrainless/car_data_pipeline/blob/main/Pipeline.png)
### 1. Data Ingestion (Bronze Layer)
- 📥 Source data loaded from Git repository to Azure SQL Database
- ⏱️ Incremental loading pattern using watermark technique:
  - `last_load`: Retrieves timestamp of last successful load
  - `current_load`: Captures current execution timestamp
  - Data between these timestamps is processed for Change Data Capture (CDC)
- 📦 Raw data stored in ADLS Gen2 as Parquet files in the Bronze container
- 🔄 Stored procedure updates watermark table after successful copy to track last processed data

### 2. Data Transformation (Silver Layer)
- 📊 Data loaded from Bronze container into Databricks
- 🧹 Transformations applied:
  ```python
  # Extract model category from Model_ID
  df = df.withColumn('model_category', split(col('Model_ID'), '-')[0])
  
  # Calculate revenue per unit
  df = df.withColumn('RevPerUnit', col('Revenue')/col('Units_Sold'))
  ```
- 🔍 Ad-hoc analysis for validation
- 🥈 Processed data stored in ADLS Gen2 Silver container

### 3. Data Modeling (Gold Layer)
- 📐 Star schema implementation with dimension and fact tables
- 🔀 Parallel processing of dimension tables:
  - `dim_model`: Extracted from Model_ID with model_category
  - `dim_dealer`: Dealer information
  - `dim_branch`: Branch information
  - `dim_date`: Date dimension
- 📊 Fact table creation by joining all dimensions with measures:
  ```python
  df_fact = df_silver.join(df_branch, df_silver.Branch_ID == df_branch.Branch_ID, how='left')
      .join(df_dealer, df_silver.Dealer_ID == df_dealer.Dealer_ID, how='left')
      .join(df_date, df_silver.Date_ID == df_date.Date_ID, how='left')
      .join(df_model, df_silver.Model_ID == df_model.Model_ID, how='left')
      .select(df_branch.dim_branch_key, df_dealer.dim_dealer_key, 
              df_model.dim_model_key, df_date.dim_date_key, 
              df_silver.Revenue, df_silver.Units_Sold, df_silver.RevPerUnit)
  ```
- 🛡️ Slowly Changing Dimension (SCD) handling:
  - Identification of new vs. existing records
  - Surrogate key generation using `monotonically_increasing_id()`
  - Proper merge operations for updates and inserts
- 🔄 Parameterized approach for both initial and incremental loads:
  ```python
  # Conditional logic for initial vs. incremental runs
  if (incremental_flag == '0'):
      max_value = 1  # Initial load
  else:
      max_value = (spark.sql("select max(dim_key) from table").collect()[0][0]) + 1
  ```
- 🔁 Delta Lake merge operations for upsert patterns:
  ```python
  # Incremental Run with Delta Lake merge
  if spark.catalog.tableExists('cars_catalog.gold.dim_table'):
      delta_tbl = DeltaTable.forPath(spark, 'abfss://gold@cardatalake00.dfs.core.windows.net/dim_table')
      delta_tbl.alias('trg').merge(df_final.alias('src'), 
                                  "trg.dim_key = src.dim_key")
               .whenMatchedUpdateAll()
               .whenNotMatchedInsertAll()
               .execute()
  ```
- 🥇 All tables registered in Unity Catalog (`cars_catalog.gold.*`) for governance and discoverability

## 🛠️ Technologies Used
- ⚙️ Azure Data Factory
- 💾 Azure SQL Database
- 🌊 Azure Data Lake Storage Gen2
- 🔥 Azure Databricks
- 📚 Databricks Unity Catalog
- 🔄 Delta Lake
- 📝 SQL & PySpark

## 🔑 Key Features

### 📈 Medallion Architecture
This project follows the modern medallion architecture pattern:
- **Bronze Layer**: Raw data preserved in its original form
- **Silver Layer**: Cleansed and transformed data with business logic applied
- **Gold Layer**: Dimensional model optimized for analytics and reporting

### 🔄 Incremental Processing
- Configurable initial and incremental load patterns
- Watermark-based tracking of processed data
- Stored procedure for metadata updates

### 📊 Dimensional Modeling
- Star schema implementation with surrogate keys
- Proper SCD (Slowly Changing Dimension) handling
- Fact table with measures and dimension keys

### 🔐 Governance & Management
- Unity Catalog for centralized metadata management
- Delta Lake for ACID transactions and time travel
- Parameterized notebooks for flexibility

## 🔮 Future Enhancements
- Add data quality validation rules
- Implement data lineage tracking
- Add real-time processing capability
- Extend dimensional model with additional business attributes
