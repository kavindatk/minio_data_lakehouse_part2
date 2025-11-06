# Setup an Iceberg, Spark, HUE, Trino, and DuckDB in MINIO Cluster

## A Containerized, Enterprise-Ready On-Premises Data Lakehouse

## 🧩 Configuration Overview

The following steps outline the complete configuration process.
I’ll be performing the setup in the order listed below to ensure everything integrates smoothly and works together as a single data lakehouse environment.

<br/><br/>

## 🔧 Step-by-Step Setup Order

### 🐘 Step 1 – Set up the PostgreSQL Database
<br/>
<i>Configure PostgreSQL to serve as the backend database for Hive Metastore and HUE.</i>
<br/><br/>
In this first step, we’ll create the init.sql file, which contains the SQL commands to initialize our PostgreSQL database.
This file will automatically create the required <b>databases and users</b> for the <b>Hive Metastore and HUE</b> services when the container starts.
<br/><br/>
After that, we’ll include this initialization script in our <b>Docker Compose</b> file and also attach a <b>data volume</b> to persist PostgreSQL data — so that your data remains safe even if the container restarts or gets rebuilt.

<br/><br/>

```sql
-- init.sql
CREATE USER hive WITH PASSWORD 'hive';
CREATE DATABASE metastore OWNER hive;
CREATE SCHEMA AUTHORIZATION hive;

CREATE USER hue WITH PASSWORD 'hue';
CREATE DATABASE hue OWNER hue;
CREATE SCHEMA AUTHORIZATION hue;

```
<br/>

```xml
  image: postgres:13
  container_name: postgres_hive
  ports:
    - "5432:5432"
  environment:
    POSTGRES_USER: root
    POSTGRES_PASSWORD: root
  volumes:
    - ./postgres/data:/var/lib/postgresql/data
    - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
```

<br/><br/>
### 🐝 Step 2 – Set up the Hive Metastore

<i>Deploy and configure Hive Metastore to store metadata for Hive, Spark, and Trino.</i>

### 🐝 Step 3 – Set up the Hive Server for Beeline Access (Optional)

<i>Install HiveServer2 to allow Beeline-based querying.</i>
<br/><br/>
### 🐝 Set Up the Hive Metastore and Hive Server
<br/> 
Now that we’ve successfully set up the <b>PostgreSQL database</b>, let’s move on to configuring the <b>Hive Metastore and Hive Server</b>.
The Hive Server will mainly be used for testing Beeline connectivity.
<br/><br/>
It’s important to note that, unlike a traditional Hadoop ecosystem, in the <b>MinIO-based setup</b>, the <b>Hive Server</b> runs in <b>embedded mode.</b>
This means it does <b>not bind</b> to the <b>default port 10000</b>, and therefore, <b>cannot be directly integrated with HUE.</b>
<br/><br/>
To address this limitation, I’ll use <b>Spark SQL (Spark Thrift Server) instead.</b>
Spark SQL will be integrated with HUE alongside <b>Trino</b>, allowing you to query data from multiple engines using the same interface.
<br/><br/>
In this step, I’ve made a few necessary modifications to the following configuration files and the Docker Compose setup:
<br/><br/>

```xml
core-site.xml
hive-site.xml
docker-compose.yml
```
<br/>

```xml
  environment:
    SERVICE_NAME: metastore
    DB_DRIVER: postgres
    HIVE_CONF: /opt/hive/conf
    HIVE_CONF_DIR: /opt/hive/conf
    SERVICE_OPTS: >
      -Djavax.jdo.option.ConnectionURL=jdbc:postgresql://postgres:5432/metastore
      -Djavax.jdo.option.ConnectionDriverName=org.postgresql.Driver
      -Djavax.jdo.option.ConnectionUserName=hive
      -Djavax.jdo.option.ConnectionPassword=hive
      -Dhive.metastore.uris=thrift://hive-metastore:9083
      -Dhive.metastore.warehouse.dir=s3a://warehouse/
      -Dfs.s3a.endpoint=http://minoproxy:9999
      -Dfs.s3a.access.key=minioadmin
      -Dfs.s3a.secret.key=minioadmin
      -Dfs.s3a.path.style.access=true
      -Dfs.s3a.impl=org.apache.hadoop.fs.s3a.S3AFileSystem
  volumes:
    - /opt/hive/conf/hive-site.xml:/opt/hive/conf/hive-site.xml
    - /opt/hive/conf/core-site.xml:/opt/hive/conf/core-site.xml
    - /opt/hive/jars/postgresql-42.7.3.jar:/opt/hive/lib/postgresql-42.7.3.jar
    - /opt/hive/jars/hadoop-aws-3.3.4.jar:/opt/hive/lib/hadoop-aws-3.3.4.jar
    - /opt/hive/jars/aws-java-sdk-bundle-1.12.367.jar:/opt/hive/lib/aws-java-sdk-bundle-1.12.367.jar
    - /opt/hive/jars/iceberg-api-1.7.0.jar:/opt/hive/lib/iceberg-api-1.7.0.jar
    - /opt/hive/jars/iceberg-aws-1.7.0.jar:/opt/hive/lib/iceberg-aws-1.7.0.jar
    - /opt/hive/jars/iceberg-aws-bundle-1.7.0.jar:/opt/hive/lib/iceberg-aws-bundle-1.7.0.jar
    - /opt/hive/jars/iceberg-common-1.7.0.jar:/opt/hive/lib/iceberg-common-1.7.0.jar
    - /opt/hive/jars/iceberg-core-1.7.0.jar:/opt/hive/lib/iceberg-core-1.7.0.jar
    - /opt/hive/jars/iceberg-hive-metastore-1.7.0.jar:/opt/hive/lib/iceberg-hive-metastore-1.7.0.jar
    - /opt/hive/jars/iceberg-hive-runtime-1.7.0.jar:/opt/hive/lib/iceberg-hive-runtime-1.7.0.jar
    - /opt/hive/jars/iceberg-orc-1.7.0:/opt/hive/lib/iceberg-orc-1.7.0  
    - /opt/hive/jars/iceberg-parquet-1.7.0.jar:/opt/hive/lib/iceberg-parquet-1.7.0.jar       
    - /opt/hive/hive_home:/home/hive
```
<br/><br/>
### 💡 Step 4 – Set up HUE

<i>Deploy HUE to provide a web-based interface for SQL querying and monitoring.</i>

Now it’s time to configure HUE — the graphical interface for querying and managing our data warehouse.
<br/><br/>
HUE requires several configuration changes to meet our setup requirements.
All of these settings are defined in the hue.ini file, which controls how HUE connects to Hive, Spark, Trino, and other services in the cluster.
<br/><br/>
In this step, we’ll focus on updating the necessary parameters inside hue.ini to make <b>HUE work seamlessly with our MinIO-based Hive Metastore, Spark SQL (Thrift Server), and Trino services.</b>
<br/><br/>

```xml
hue:
  image: gethue/hue:latest
  container_name: hue
  platform: linux/amd64
  ports:
    - "8888:8888"
  networks:
    - data_lake_net
  environment:
    - HUE_CONF_DIR=/usr/share/hue/desktop/conf
  volumes:
    - /opt/hue/conf/hue.ini:/usr/share/hue/desktop/conf/hue.ini
```
<br/><br/>

## 🚀 Start the Services
<br/><br/>
So far, we have successfully set up <b>Hive and HUE with a PostgreSQL backend.</b>
Once all configurations are complete, your final folder structure should look similar to this (you can verify that all necessary files and configuration folders are in place).
<br/><br/>

```bash
hadoop@node01:/opt$ tree /opt/
/opt/
├── hive
│   ├── conf
│   │   ├── core-site.xml
│   │   ├── hive-site.xml
│   │   └── log4j2.properties
│   ├── docker-compose.yml
│   ├── hive_home
│   ├── jars
│   │   ├── aws-java-sdk-bundle-1.12.367.jar
│   │   ├── hadoop-aws-3.3.4.jar
│   │   ├── iceberg-api-1.7.0.jar
│   │   ├── iceberg-aws-1.7.0.jar
│   │   ├── iceberg-aws-bundle-1.7.0.jar
│   │   ├── iceberg-common-1.7.0.jar
│   │   ├── iceberg-core-1.7.0.jar
│   │   ├── iceberg-hive-metastore-1.7.0.jar
│   │   ├── iceberg-hive-runtime-1.7.0.jar
│   │   ├── iceberg-orc-1.7.0
│   │   ├── iceberg-orc-1.7.0.jar
│   │   ├── iceberg-parquet-1.7.0.jar
│   │   └── postgresql-42.7.3.jar
│   ├── postgres
│       ├── data  
│       └── init.sql
├── hue
    ├── conf
        └── hue.ini
```

<br/><br/>

Now it’s time to <b>start the services.</b>, First Navigate into the <b>Hive</b> folder
<br/>

[Hive Folder](https://github.com/kavindatk/minio_data_lakehouse_part2/tree/main/hive)

Then You can use the following command to start all the containers:
<br/><br/>

```bash
docker compose -p hive_cluster up -d

docker compose -p hive_cluster down -v # This for stop the cluster 
```

<br/>
To check the current status of your setup, run:
<br/><br/>

```
docker ps

docker logs hue # Can check the logs
```
<br/><br/>




### Step 5 – Set up the Trino Cluster with Hive and Iceberg Catalogs

<i>Configure Trino for distributed SQL queries with Hive and Iceberg integration.</i>



### Step 6 – Set up the Spark Cluster with Spark Thrift (Spark SQL) Access

<i>Enable Spark for analytics and interactive SQL queries.</i>



### Step 7 – Set up DuckDB and Perform Testing

<i>Integrate DuckDB for lightweight, fast querying and verification across datasets.</i>
