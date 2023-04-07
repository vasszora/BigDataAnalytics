# Hadoop-spark cluster on Docker

This is an example of how to run a Hadoop-spark cluster on Docker. The following picture shows the architecture of the cluster.
![alt text](pics/docker.png "architecture")

## Prerequisites

- Docker engine
- Docker compose
- Python and pyspark module

## How to run

- Clone the repository
- Run the following command to start the cluster

```shell
cd BigDataAnalytics
docker build -t hadoop-spark .
docker-compose up -d
```

This will start the nodes and creates a bridge network called `hadoop-spark-net`. The nodes will be accessible through the following hostname:

- 'master'
- 'worker1'

Also it starts the ssh server on each of the nodes, and the hadoop and spark services.

## How to use

- To access the master node, run the following command

```shell
docker exec -it master bash
```

## Web UI

- Hadoop web UI: <http://localhost:9870>
- Spark web UI: <http://localhost:8080>

## HDFS

- To create a directory in HDFS, run the following command

```shell
hduser@master$hdfs dfs -mkdir /taltech
```

- To copy a file from local file system to HDFS, run the following command

```shell
hduser@master$hdfs dfs -put /home/hduser/README.md /taltech
```

- To copy a file from HDFS to local file system, run the following command

```shell
hduser@master$hdfs dfs -get /taltech/README.md /home/hduser/README2.md
```

- To list the files in HDFS, run the following command

```shell
hduser@master$hdfs dfs -ls /taltech
```

## YARN

- To run a spark job, run the following command

```shell
hduser@master$yarn jar /home/hduser/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.2.jar wordcount /taltech/README.txt /taltech/rst 
```

- To check the status of the job, run the following command

```shell
hduser@master$yarn app -list
```

- Kill an application

```shell
hduser@master$yarn app -kill *ID*
```

- Print the node list

```shell
hduser@master$yarn node -list
```

## Spark

- To start an interactive pyspark shell, run the following command

```shell
hduser@master$pyspark
```

- Create and RDD from a file

```python
rdd = sc.textFile("/home/hduser/README.md")
```

- Create from a hdfs file

```python
rdd = sc.textFile("hdfs://master:9000/taltech/README.txt")
```

- Count the number of lines

```python
rdd.count()
```

- Dataframe from list

```python
df = spark.createDataFrame([("a", 1), ("b", 2), ("c", 3)], ["letter", "number"])
```

- Schema of the dataframe

```python
df.printSchema()
```

- Show the dataframe

```python
df.show()
```

- Create a dataframe from a csv file

```python
df = spark.read.csv('/home/hduser/flight.csv', header=True)
```

## Connect with a Postgresql container

- Add a new service to the docker-compose.yml file
```yml
  db:
    image: postgres:14.1-alpine
    container_name: postgres
    hostname: postgres
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_USER=postgres
    volumes:
      - db:/var/lib/postgresql/data
```

- Add volume for db
```yml
volumes:
  db:
    driver: local
```
- Download Postgresql driver and move it to your workspace ()
- Copy the driver file to the container in the Dockerfile
```Dockerfile
COPY postgresql-42.6.0.jar /home/hduser/
```
- Since the Dockerfile changed, rebuild the hadoop_spark image
```shell
docker build -t hadoop-spark .
```
- You can check if the copy was successful
```shell
docker-compose up -d
docker exec -it master bash

hduser@master$hdfs ls
hadoop  postgresql-42.6.0.jar  spark
```

- Add a table and some data to the database
```sql
docker exec -it postgres psql -U postgres

postgres=# CREATE TABLE users (user_id serial PRIMARY KEY, name TEXT);
postgres=# INSERT INTO users(name) VALUES ('Easter bunny');
postgres=# INSERT INTO users(name) VALUES ('Santa Claus');
```


- Open localhost:8888 and create a new Jupyter notebook
- Create a SparkSession, and specify the postgresql jar file with the config
```python
from pyspark.sql import SparkSession

spark = SparkSession \
    .builder \
    .appName("Python Spark SQL basic example") \
    .config("spark.jars", "/home/hduser/postgresql-42.6.0.jar") \
    .getOrCreate()
```
- Read the created table from the database
```python
df = spark.read \
    .format("jdbc") \
    .option("url", "jdbc:postgresql://postgres:5432/postgres") \
    .option("dbtable", "users") \
    .option("user", "postgres") \
    .option("password", "postgres") \
    .option("driver", "org.postgresql.Driver") \
    .load()
```
- Print the schema
```python
df.printSchema()

root
 |-- user_id: integer (nullable = true)
 |-- name: string (nullable = true)
```

## Connect with a MongoDB container
- Build hadoop_spark image
```shell
docker build -t hadoop-spark .
```
- Start the containers
```shell
docker-compose up -d
```
- Add a table and some data to the database
```sql
docker exec -it mongodb mongosh -u user -p pass

test> db.createCollection("users")
test> db.users.insert({"name":"Santa Claus"})
```

- Open localhost:8888 and create a new Jupyter notebook
- Create a SparkSession with configurations added for the connection with MongoDb
```python
from pyspark.sql import SparkSession

spark = SparkSession \
    .builder \
    .appName("Python Spark SQL basic example") \
    .config("spark.mongodb.read.connection.uri", "mongodb://user:pass@mongodb") \
    .config("spark.mongodb.read.collection", "users") \
    .config("spark.mongodb.read.database", "test") \
    .getOrCreate()
  
df = spark.read.format("mongodb") \
    .option("user", "user") \
    .option("password", "pass") \
    .load()
  
df.printSchema()

root
 |-- _id: string (nullable = true)
 |-- name: string (nullable = true)
 
```

## Jupyter notebook and vscode

You can connect the jupyter notebook and vscode to the master node. To do this please follow the following steps:

1. Install the python extension in vscode
2. Install the jupyter extension in vscode
3. CTRL+SHIFT+P and select the command "Jupyter: Specify Jupyter server for connections"
4. Select the "Existing" option
5. Enter the following url: <http://localhost:8888>
6. Enter a name for the connection
7. Open the example notebook in the jupyter notebook change the kernel to the one you created in the previous step and run the cells

Problem with the jupyter notebook? <https://github.com/nteract/hydrogen/issues/922#issuecomment-405456346>: try to open the <http://localhost:8888> in the browser and create a new notebook. Then you can connect to the notebook from vscode.
