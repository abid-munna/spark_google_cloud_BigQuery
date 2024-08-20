# SparkGcBigQuery
Project Description:Breadcrumbsspark_google_cloud_BigQuery Data Pipeline Project Overview:  This project aims to establish a robust and scalable data pipeline that efficiently ingests, processes, and stores data to Google Cloud. develop a data pipeline using Apache-Spark as the processing tool and the  Google Cloud Storage (raw file storage) and Google BigQuery (serve the processed data to used for analytical queries) as the storage solution.



#### Run the given code to get data
get all data as zip file using the url or use request library with necessory ssl certficates
```https://download.inep.gov.br/microdados/microdados_censo_da_educacao_superior_{year}.zip"

```
run below script for unzip the file to data folder
```
# run the script environment
python3 get_data.py

```
 
### 2. Prepare Docker-Compose File , DockerFile and gc_cedential


We use "volumes" to import our scripts to containers.
      - ./data:/data
      -./src:/src

See ```docker-compose.yml```

```
version: '3'

services:
  spark:
    build: .
    environment:
      - SPARK_MODE=master
    ports:
      - '8080:8080'
      - '4040:4040'
    volumes:
      - ./data:/data
      - ./src:/src
  spark-worker:
    build: .
    environment:
      - SPARK_MODE=worker
      - SPARK_MASTER_URL=spark://spark:7077
      - SPARK_WORKER_MEMORY=4G
      - SPARK_EXECUTOR_MEMORY=4G
      - SPARK_WORKER_CORES=4
    volumes:
      - ./data:/data
      - ./src:/src 

```

Dockerfile add gc_credntial.json to ENV variable
```FROM docker.io/bitnami/spark:3.3.1

COPY *.jar $SPARK_HOME/jars

RUN mkdir -p $SPARK_HOME/secrets
COPY ./src/credentials/gcp-credentials.json $SPARK_HOME/secrets/gcp-credentials.json
ENV GOOGLE_APPLICATION_CREDENTIALS=$SPARK_HOME/secrets/gcp-credentials.json

RUN pip install delta-spark
```
use delta-spark use to make the spark - cloud storage connection

### 3. Running docker-compose file
Open your workspace folder which includes all files provided and run the given command as below.
```
# run docker-compose file
docker-compose up --build
```
After the container is running, you can set up your environment.

#### Insert to Delta Table Google storage
 
```
# Execute kafka container with container id given above
docker exec -it <spark_container name> bash

# Create Kafka "odometry" topic for ROS odom data
spark-submit --packages io.delta:delta-core_2.12:2.1.0 --master spark://spark:7077 /src/insert_csv_into_delta_table_gcs.py
```
#### Process Delta table to BigQury Table
```
# inside container
spark-submit --packages io.delta:delta-core_2.12:2.1.0,com.google.cloud.spark:spark-3.1-bigquery:0.28.0-preview /src/aggregate_delta_gcs_to_gbq_table.py
```
### BigQuery Execute Query to perfom an an analysis
```
SELECT
  CO_CINE_AREA_GERAL,
  NO_CINE_AREA_GERAL,
  SUM(QT_ING_MASC)/SUM(QT_ING_FEM + QT_ING_MASC) AS PERCENT_MASC,
  SUM(QT_ING_FEM)/SUM(QT_ING_FEM + QT_ING_MASC) AS PERCENT_FEM  
FROM 
  `<bucket>.<table name>`
GROUP BY
  NO_CINE_AREA_GERAL,
  CO_CINE_AREA_GERAL
ORDER BY
  PERCENT_MASC
```

