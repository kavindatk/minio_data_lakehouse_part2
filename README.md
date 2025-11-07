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

### 🚀 Start the Services
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




### ⚡ Step 5 – Set up the Trino Cluster with Hive and Iceberg Catalogs

<i>Configure Trino for distributed SQL queries with Hive and Iceberg integration.</i>
<br/><br/>
Now that our Hive setup is up and running (and we’ve also enabled Iceberg), it’s time to move on to the Trino cluster configuration.
<br/><br/>
In my setup:
<br/><br/>
1. <b>VM1</b> acts as the <b>Trino Coordinator</b>
2. <b>VM2, VM3, and VM4</b> act as the <b>Worker nodes</b>


<br/>

One important point to note — I’ve configured the <b>Trino cluster</b> to use the <b>host network mode.</b>
This allows seamless communication between all VMs without any network-related issues or Docker network limitations.

<br/>

#### <b>Trino Coordinator</b>

<br/>

```bash
hadoop@node01:/opt$ tree trino/
trino/
├── data
│   ├── etc -> /etc/trino
│   ├── plugin -> /usr/lib/trino/plugin
│   ├── secrets-plugin -> /usr/lib/trino/secrets-plugin
│   └── var   
├── docker-compose.yml
└── etc
    ├── catalog
    │   ├── hive.properties
    │   ├── iceberg.properties
    │   └── postgresql.properties
    ├── config.properties
    ├── hadoop
    ├── jvm.config
    ├── node.properties
    └── password.properties

```
<br/>

[Trino Coordinator](https://github.com/kavindatk/minio_data_lakehouse_part2/tree/main/trino_coordinator)

<br/>

#### <b>Trino Worker</b>

<br/>

```bash
hadoop@node02:/opt$ tree trino/
trino/
├── data
├── docker-compose.yml
└── etc
    ├── catalog
    │   ├── hive.properties
    │   ├── iceberg.properties
    │   └── postgresql.properties
    ├── config.properties
    ├── hadoop
    ├── jvm.config
    └── node.properties

```
<br/>

[Trino Worker](https://github.com/kavindatk/minio_data_lakehouse_part2/tree/main/trino_worker)

<br/>

Once the configuration is complete, we’ll start the Trino services on all nodes and verify that the cluster is connected and running correctly.

Then You can use the following command to start all the containers:
<br/>

```bash
docker compose -p trino_cluster up -d

docker compose -p trino_cluster down -v # This for stop the cluster 
```

<br/>
To check the current status of your setup, run:
<br/><br/>

```
docker ps

docker logs trino_service # Can check the logs
```
<br/><br/>

Once everything is up and running correctly, you can verify the Trino cluster by accessing the following web link:

```bash
http://<coordinator-host>:8080/ui/
```
<br/><br/>


### 🔥 Step 6 – Set up the Spark Cluster with Spark Thrift (Spark SQL) Access

<i>Enable Spark for analytics and interactive SQL queries.</i>

<br/>

Now it’s time for the big one — <b>Apache Spark.</b>
<br/><br/>
In this setup, I’ll configure <b>VM1</b> as the <b>Spark Master</b> and also as the <b>Spark Thrift Server.</b>
<br/><br/>
The other nodes — <b>VM2, VM3, and VM4</b> — will be configured as <b>Spark Workers.</b>
<br/>
<br/><br/>
It’s important to note that <b>worker nodes don’t require Thrift Server configurations;</b> the <b>Master node</b> will handle all coordination and job execution.
<br/><br/>
In this setup, each of the three worker VMs will host <b>two worker</b> processes, resulting in a <b>final configuration of one Spark Master with six workers.</b>
<br/><br/>

One important point to note — I’ve configured the <b>Spark cluster</b> to use the <b>host network mode.</b>
This allows seamless communication between all VMs without any network-related issues or Docker network limitations.

<br/>

#### <b>Spark Master and Spark Thrift Server</b>

<br/>

```bash
hadoop@node01:~$ tree /opt/spark/
/opt/spark/
├── conf
│   ├── core-site.xml
│   ├── hive-site.xml
│   ├── log4j2.properties
│   ├── spark-defaults.conf
│   └── workers
├── docker-compose.yml
└── jars
    ├── aws-java-sdk-bundle-1.12.367.jar
    ├── hadoop-aws-3.3.4.jar
    ├── iceberg-aws-bundle-1.10.0.jar
    ├── iceberg-spark-runtime-4.0_2.13-1.10.0.jar
    └── postgresql-42.7.3.jar

```
<br/>

[Spark Master and Spark Thrift Server](https://github.com/kavindatk/minio_data_lakehouse_part2/tree/main/spark_master)

<br/>

#### <b>Spark Worker</b>

<br/>

```bash
hadoop@node02:~$ tree /opt/spark/
/opt/spark/
├── conf
│   ├── log4j2.properties
│   ├── spark-defaults.conf
│   └── workers
├── docker-compose.yml
└── jars
    ├── aws-java-sdk-bundle-1.12.367.jar
    ├── hadoop-aws-3.3.4.jar
    ├── iceberg-aws-bundle-1.10.0.jar
    ├── iceberg-spark-runtime-4.0_2.13-1.10.0.jar
    └── postgresql-42.7.3.jar

```
<br/>

[Spark Worker](https://github.com/kavindatk/minio_data_lakehouse_part2/tree/main/spark_worker)

<br/>

Once the configuration is complete, we’ll start the Spark services on all nodes and verify that the cluster is connected and running correctly.

Then You can use the following command to start all the containers:
<br/>

```bash
docker compose -p spark_cluster up -d

docker compose -p spark_cluster down -v # This for stop the cluster 
```

<br/>
To check the current status of your setup, run:
<br/><br/>

```
docker ps

docker logs spark_master # Can check the logs
docker logs spark_thrift # Can check the logs
docker logs spark_worker_1 # Can check the logs
docker logs spark_worker_2 # Can check the logs
```
<br/><br/>

Once everything is up and running correctly, you can verify the Spark cluster by accessing the following web link:

```bash
http://<master-host>:8088
```
<br/>
Once everything is up , you can try on pyspark using following commands 

```bash
# From master 
hadoop@node01:~$ docker exec -it spark_master /opt/spark/bin/pyspark --master spark://node01:7077

Python 3.10.12 (main, Aug 15 2025, 14:32:43) [GCC 11.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
WARNING: Using incubator modules: jdk.incubator.vector
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /__ / .__/\_,_/_/ /_/\_\   version 4.0.0
      /_/

Using Python version 3.10.12 (main, Aug 15 2025 14:32:43)
Spark context Web UI available at http://node01:4041
Spark context available as 'sc' (master = spark://node01:7077, app id = app-20251107071025-0002).
SparkSession available as 'spark'.

# From Worker

hadoop@node02:/opt$ docker exec -it spark_worker_1 /opt/spark/bin/pyspark --master spark://node01:7077

Python 3.10.12 (main, Aug 15 2025, 14:32:43) [GCC 11.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
WARNING: Using incubator modules: jdk.incubator.vector
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /__ / .__/\_,_/_/ /_/\_\   version 4.0.0
      /_/

Using Python version 3.10.12 (main, Aug 15 2025 14:32:43)
Spark context Web UI available at http://node02:4040
Spark context available as 'sc' (master = spark://node01:7077, app id = app-20251107071157-0003).
SparkSession available as 'spark'.

```
<br/><br/>
### 💻 Accessing the Cluster via HUE
<br/>

So far, we have successfully set up Spark, Trino, and Hive.
Once all the services are up and running, you can access them through the HUE interface.
<br/>
```bash
http://<hue-host>:8888/hue
```
<br/>
When you first log in to HUE, you may see a Hive error message — this happens because the HiveServer (Thrift service) is not running.
As mentioned earlier, we are not using Hadoop in this setup; instead, we are using MINIO as the storage backend.
Since the HiveServer Thrift port (10000) isn’t active, HUE shows this connection error.
<br/><br/>
However, you can still use SparkSQL within HUE to get a similar experience to Hive.
Remember, Hive usually runs on top of MapReduce, Tez, or Spark execution engines — in our case, it runs on Spark.
<br/><br/>

### 🐤 Step 7 – Set up DuckDB and Perform Testing

<i>Integrate DuckDB for lightweight, fast querying and verification across datasets.</i>
<br/><br/>
Unlike above Spark,Hive and HUE tools, <b>DuckDB</b> is lightweight and very easy to install.
Currently, DuckDB works in <b>standalone mode</b>, And for my case I’m not using Docker for this setup.
With just a single command, you can install DuckDB on a Linux system.
<br/><br/>
The following steps show how to <b>install and configure DuckDB</b>, and then <b>access data stored in MINIO.</b>
<br/><br/>

1. Install DuckDB (internet connection required)

<br/><br/>
```bash
curl https://install.duckdb.org | sh
```
<br/> <br/>  

2. Configure DuckDB
   
<br/><br/>
```bash
nano ~/.bashrc

# Add following content

#DuckDB
export PATH='/home/hadoop/.duckdb/cli/latest':$PATH

source ~/.bashrc
```
<br/><br/>

3. Access MINIO data using DuckDB
<br/><br/>

```bash
duckdb
```


   

