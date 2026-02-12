# Data Engineering Streaming Project

A real-time data pipeline using **Kafka** for streaming, **Spark** for processing, **Airflow** for orchestration, and **Docker** for containerization.

Data is fetched from the [Random Name API](https://randomuser.me/), streamed to a Kafka topic via a Python script, orchestrated by an Airflow DAG, and processed with Spark Structured Streaming.

---

## Architecture

```
Random Name API → Python Script → Kafka Producer → Spark Structured Streaming → Output
                        ↑
                   Airflow DAG (orchestration)
```

All services run in isolated Docker containers for modularity and scalability.

---

## Prerequisites

- **Docker**: [Install Docker Desktop](https://www.docker.com/)
- **Python 3.x**

## Quick Start

```bash
git clone https://github.com/simardeep1792/Data-Engineering-Streaming-Project.git
cd Data-Engineering-Streaming-Project

docker network create docker_streaming
docker-compose -f docker-compose.yml up -d
```

---

## Project Structure

### `docker-compose.yml`
Defines all services (Compose v3.7):

| Service | Description |
|---------|-------------|
| **Airflow** | PostgreSQL DB + Web Server with admin user setup |
| **Kafka** | Zookeeper + 3 Brokers + Connect + Schema Registry + UI |
| **Spark** | Master node for data processing |

**Networks:** `kafka_network` (Kafka-dedicated) and `default` (`docker_streaming`).

### `kafka_stream_dag.py`
Airflow DAG (`name_stream_dag`) that runs daily at 1 AM. Triggers the `initiate_stream` function to fetch and publish data to Kafka. Configured with `catchup=False` and `max_active_runs=1`.

### `kafka_streaming_service.py`
Handles the data pipeline logic:
- Fetches random user data from the API
- Transforms and formats data for Kafka (includes zip code hashing for privacy)
- Configures a Kafka producer and publishes messages
- Runs on a configurable interval for the set streaming duration

### `spark_processing.py`
Consumes data from Kafka and processes it:
- Initializes a Spark session
- Reads a streaming DataFrame from the Kafka topic
- Transforms raw data into a structured format
- Writes output in Parquet format with checkpointing for data integrity

---

## Step-by-Step Pipeline Setup

### 1. Start Kafka Cluster
```bash
docker network create docker_streaming
docker-compose -f docker-compose.yml up -d
```

### 2. Create Kafka Topic
Go to the Kafka UI at `http://localhost:8888/` → **Topics** → Create **`names_topic`** with replication factor **3**.

### 3. Configure Airflow
```bash
docker-compose run airflow_webserver airflow users create \
  --role Admin --username admin --email admin \
  --firstname admin --lastname admin --password admin
```

### 4. Install Dependencies & Validate DAGs
```bash
./airflow.sh bash
pip install -r ./requirements.txt
airflow dags list
```

### 5. Start Airflow Scheduler
```bash
airflow scheduler
```

### 6. Verify Data in Kafka
Check `http://localhost:8888/` to confirm messages are flowing into `names_topic`.

### 7. Run Spark Processing
```bash
docker cp spark_processing.py spark_master:/opt/bitnami/spark/
docker exec -it spark_master /bin/bash

# Download required JARs
cd jars
curl -O https://repo1.maven.org/maven2/org/apache/kafka/kafka-clients/2.8.1/kafka-clients-2.8.1.jar
curl -O https://repo1.maven.org/maven2/org/apache/spark/spark-sql-kafka-0-10_2.13/3.3.0/spark-sql-kafka-0-10_2.13-3.3.0.jar
curl -O https://repo1.maven.org/maven2/org/apache/commons/commons-pool2/2.8.0/commons-pool2-2.8.0.jar

# Submit Spark job
cd ..
spark-submit \
  --master local[2] \
  --jars jars/kafka-clients-2.8.1.jar,jars/spark-sql-kafka-0-10_2.13-3.3.0.jar,jars/commons-pool2-2.8.0.jar \
  spark_processing.py
```

---

## Common Issues

| Issue | Fix |
|-------|-----|
| Services won't start | Verify `docker-compose.yml` env vars and configs |
| Service dependency errors | Ensure correct startup order (Zookeeper before Kafka brokers) |
| DAG not showing in Airflow | Check `kafka_stream_dag.py` for syntax/logic errors |
| Spark job fails | Confirm all required JARs are downloaded and compatible |
| Kafka topic issues | Verify replication factor matches broker count |
| Docker networking errors | Ensure `docker_streaming` network exists and services are on correct networks |
| Deprecation warnings | Non-blocking; update methods in future iterations |
