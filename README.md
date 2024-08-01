# Handling Large JSON Data with Kafka, Spark Streaming, and Flask API

This guide details how to handle large JSON data (up to 5MB) using Kafka for ingestion, Spark Streaming for processing, and Flask for serving computed KPIs.

## Prerequisites

- Apache Kafka and Zookeeper
- Apache Spark
- Python environment with necessary libraries

## Step 1: Kafka Setup

1. **Install Kafka and Zookeeper**:
   - Follow the [Kafka Quickstart guide](https://kafka.apache.org/quickstart) to download and start Kafka and Zookeeper.

2. **Create Kafka Topics**:
   - Create a topic for incoming JSON data (`device-data`).

    ```sh
    bin/kafka-topics.sh --create --topic device-data --bootstrap-server localhost:9092 --partitions 10 --replication-factor 1
    ```

## Step 2: Spark Streaming Application

1. **Install Required Libraries**:
   - Ensure you have the necessary Python libraries installed.

    ```sh
    pip install pyspark kafka-python
    ```

2. **Spark Streaming Code**:
   - Create a Python script `spark_streaming.py` to process the stream data from Kafka and store processed data in temporary storage.

    ```python
    from pyspark.sql import SparkSession
    from pyspark.sql.functions import from_json, col, avg
    from pyspark.sql.types import StructType, StructField, StringType, TimestampType, FloatType
    import os

    # Define the schema of the device data
    schema = StructType([
        StructField("device_id", StringType(), True),
        StructField("timestamp", TimestampType(), True),
        StructField("value", FloatType(), True)
    ])

    # Initialize Spark Session
    spark = SparkSession.builder \
        .appName("Device Data KPI Computation") \
        .getOrCreate()

    # Read streaming data from Kafka
    df = spark \
        .readStream \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "localhost:9092") \
        .option("subscribe", "device-data") \
        .load()

    # Parse the JSON data
    df = df.selectExpr("CAST(value AS STRING) as json") \
        .select(from_json(col("json"), schema).alias("data")) \
        .select("data.*")

    # Filter data for a specific timestamp t1 (you can customize this condition)
    t1 = "2024-07-30T12:00:00Z"
    filtered_df = df.filter(col("timestamp") == t1)

    # Compute the average value as KPI
    kpi_df = filtered_df.groupBy("timestamp").agg(avg("value").alias("average_value"))

    # Define a function to write the KPI results to a temporary location
    def write_to_temp_location(df, epoch_id):
        # Define the directory for temporary storage
        temp_dir = "/path/to/temp/directory"
        
        # Convert Spark DataFrame to Pandas DataFrame and save as CSV
        pandas_df = df.toPandas()
        file_path = os.path.join(temp_dir, f"kpi_{epoch_id}.csv")
        pandas_df.to_csv(file_path, index=False)

    # Write the KPI results to the temporary location
    query = kpi_df.writeStream \
        .foreachBatch(write_to_temp_location) \
        .outputMode("complete") \
        .start()

    # Await termination
    query.awaitTermination()
    ```

## Step 3: API Service to Retrieve KPIs

1. **Install Flask**:
   - Install Flask for the API service.

    ```sh
    pip install Flask
    ```

2. **API Service Code**:
   - Create a Python script `api_service.py` to provide an endpoint to retrieve the KPIs.

    ```python
    from flask import Flask, jsonify
    import os
    import pandas as pd

    app = Flask(__name__)

    # Define the directory for temporary storage
    temp_dir = "/path/to/temp/directory"

    def get_kpis(timestamp):
        # Define the file path pattern
        file_pattern = os.path.join(temp_dir, f"kpi_*.csv")
        
        # List all files in the temporary directory
        files = [f for f in os.listdir(temp_dir) if f.startswith("kpi_")]
        
        # Read the CSV files and filter by timestamp
        for file in files:
            df = pd.read_csv(os.path.join(temp_dir, file))
            filtered_df = df[df['timestamp'] == timestamp]
            if not filtered_df.empty:
                return filtered_df.to_dict(orient='records')
        
        return []

    @app.route('/kpis/<timestamp>', methods=['GET'])
    def kpis(timestamp):
        results = get_kpis(timestamp)
        return jsonify(results)

    if __name__ == '__main__':
        app.run(port=5000, debug=True)
    ```

## Step 4: Handling Large JSON Data in Kafka

To handle large JSON data in Kafka, you might need to adjust the Kafka configuration to allow larger message sizes. Here’s how you can do it:

1. **Increase the Maximum Message Size**:
   - Update the Kafka broker configuration (`server.properties`) to increase the maximum message size.

    ```sh
    # server.properties
    message.max.bytes=10485760  # 10MB
    ```

2. **Update Kafka Producer Configuration**:
   - Ensure that the Kafka producer configuration allows for larger messages.

    ```python
    from kafka import KafkaProducer
    import json

    producer = KafkaProducer(
        bootstrap_servers='localhost:9092',
        value_serializer=lambda v: json.dumps(v).encode('utf-8'),
        max_request_size=10485760  # 10MB
    )

    # Example of sending a large JSON message
    large_json = {"device_id": "device123", "timestamp": "2024-07-30T12:00:00Z", "value": 42.0}
    producer.send('device-data', large_json)
    ```

## Running the Setup

1. **Start Kafka and Zookeeper**:
   - Follow the Kafka Quickstart guide to start Kafka and Zookeeper.

2. **Run the Spark Streaming Application**:
   - Execute the `spark_streaming.py` script to start processing the incoming data.

    ```sh
    spark-submit --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.0.1 spark_streaming.py
    ```

3. **Run the API Service**:
   - Execute the `api_service.py` script to start the API service.

    ```sh
    python api_service.py
    ```

4. **Test the Setup**:
   - Send JSON data to the Kafka topic `device-data`.
   - Use a tool like Postman or curl to query the API for KPIs.

    ```sh
    curl http://localhost:5000/kpis/2024-07-30T12:00:00Z
    ```

This setup ensures that large JSON data is streamed, processed, and stored efficiently, with quick access to the results through an API. By increasing Kafka's message size limit and using temporary storage for processed results, you can handle large payloads and provide fast access to computed KPIs.
