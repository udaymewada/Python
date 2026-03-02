Explanation of all PySpark RDD, DataFrame and SQL examples present on this project are available at [Apache PySpark Tutorial](https://sparkbyexamples.com/pyspark-tutorial/), All these examples are coded in Python language and tested in our development environment.

# Table of Contents (Spark Examples in Python)

## Getting Started

Follow these steps to set up a local environment that can run the example scripts:

1. **Install prerequisites**
   - Python 3.8 or later.
   - Java 8 or later (required by Spark). Ensure `JAVA_HOME` points to your JDK installation.
2. **Create a virtual environment (recommended)**
   ```bash
   python -m venv .venv
   source .venv/bin/activate  # On Windows use: .venv\\Scripts\\activate
   ```
3. **Install dependencies**
   ```bash
   pip install --upgrade pip
   pip install pyspark pandas
   ```
   Pandas is optional but useful for interoperability examples.
4. **Verify Spark can start**
   ```bash
   python - <<'PY'
   from pyspark.sql import SparkSession
   spark = SparkSession.builder.appName("check").getOrCreate()
   print(spark.version)
   spark.stop()
   PY
   ```
   Seeing the Spark version confirms that Java and PySpark are configured correctly.

## Running the examples

All examples are single-file scripts. After activating your virtual environment:

- Run any script directly, for example:
  ```bash
  python pyspark-sparksession.py
  ```
- Scripts that read sample data expect files in the `resources/` directory (for example `resources/zipcodes.csv`) or the root-level `data.txt`. Run them from the repository root so relative paths resolve correctly.
- To explore or modify an example, open the corresponding `.py` file and adjust the DataFrame operations as needed.
- If you prefer the interactive shell, you can copy transformations from a script into `pyspark` or a notebook after creating a `SparkSession`.

# PySpark Basic Examples
- How to create SparkSession
- PySpark – Accumulator
- PySpark Repartition vs Coalesce
- PySpark Broadcast variables
- PySpark – repartition() vs coalesce()
- PySpark – Parallelize
- PySpark – RDD
- PySpark – Web/Application UI
- PySpark – SparkSession
- PySpark – Cluster Managers
- PySpark – Install on Windows
- PySpark – Modules & Packages
- PySpark – Advantages
- PySpark – Feature
- PySpark – What is it? & Who uses it?


## PySpark DataFrame Examples 
- PySpark – Create a DataFrame
- PySpark – Create an empty DataFrame
- PySpark – Convert RDD to DataFrame
- PySpark – Convert DataFrame to Pandas
- PySpark – StructType & StructField
- PySpark Row using on DataFrame and RDD
- Select columns from PySpark DataFrame 
- PySpark Collect() – Retrieve data from DataFrame
- PySpark withColumn to update or add a column
- PySpark using where filter function 
- PySpark – Distinct to drop duplicate rows 
- PySpark orderBy() and sort() explained
- PySpark Groupby Explained with Example
- PySpark Join Types Explained with Examples
- PySpark Union and UnionAll Explained
- PySpark UDF (User Defined Function
- PySpark flatMap() Transformation
- PySpark map Transformation


## PySpark SQL Functions
- PySpark Aggregate Functions with Examples
- PySpark Window Functions


## PySpark Datasources
- PySpark Read CSV file into DataFrame
- PySpark read and write Parquet File

