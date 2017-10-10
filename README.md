# hadoopsql

A test comparing the performance of different SQL engines in Hadoop/IBM BigInsights/HortonWorks HDP environment. The following SQL engines are used:
* Hive on text files
* Hive on Parquet files
* Hive on OCR files
* Big SQL on Hive Parquet files
* Big SQL on Hive OCR files
* Spark SQL
* Phoenix HBase SQL

The same data set is loaded into a particular surrounding and the same three SQL queries are launched and time spent to execute is taken.
No optimization is done to boost the execution of a particular engine. Default configuration out of the box is used.
It is not any kind of benchmarking and one should be extra careful to generalize the results. 

# Test scenario

1. Load data into MySQL database. Four tables are created: SALES, CUSTOMERS, EMPLOYEES, PRODUCTS
2. Import data from MySQL into Hive using Sqoop utility
3. Run queries on Hive text, Parquet and OCR
4. Catalog Hive tables into IBM BigSQL. Run queries using Big SQL engine.
5. Load data into Spark and execute queries using Spark SQL.
6. Load data into HBase Phoenix and run queries through Phoenix SQL
* There are huge differences in execution time while running the same command so I used to run the query three times one after the other and calculate the arithmetic mean.

# Queries under test

Queries are simple: aggregate over a single table, join two tables and join three tables.

1. select salespersonid, sum(quantity) from SALES group by salespersonid
2. select * from (select salespersonid, sum(quantity) from SALES group by salespersonid) as s,EMPLOYEES where salespersonid = employeeid
3. select * from (select c.*,q from (select customerid,sum(quantity) as q from SALES group by customerid) as s,CUSTOMERS as c where c.customerid = s.customerid) as r order by q desc LIMIT 20

# Download data 

```BASH
git clone https://github.com/stanislawbartkowski/hadoopsql.git
cd hadoopsql
tar xvfz data.tgz
```

# Load data into MySQL database

* Prepare MySQL (MariaDB) database. Embedded HDP MySQL database can be used.

As admin user 

```SQL
CREATE DATABASE TESTDB;
CREATE USER 'test'@'%' IDENTIFIED BY 'test';
GRANT ALL PRIVILEGES ON *.* TO 'test'@'%';
```

Launch mysql console ad test user 
```BASH
cd hadoopsql
mysql -h {host} -u test -p testdb
```

Load data

```SQL
source create.sql
```

Run queries in MySQL

# Load data into Hive

If you are running as a particular user, make sure that home HDFS directory is created. Logon as hdfs user and execute:

```BASH
hdfs dfs -mkdir /user/{username}
hdfs dfs -chown {username} /user/{username}
```
In case of simple HDFS authorization system add {user} access to hive directory (as user hdfs)
```BASH
hdfs dfs -chmod 777 /apps/hive/warehouse
```

* Customize imphive script file, provide host name for MySQL database
* Create testdb database in Hive
```SQL
create database testdb;
```
```BASH
./imphive
```
# Run queries on Hive text files

Sqoop creates Hive tables as text files. Connect to Hive engine and testdb database (schema) and execute three queries

Connect to Hive as a user:
```BASH
beeline -u "jdbc:hive2://{hive server}:10000/{username}" -n sb $@
```
```SQL
use testdb;
```
Execute three queries

# Run queries on Hive Parquet files
```SQL
create database testdb1;
use testdb1;
create table products stored as parquet as select * from testdb.products;
create table customers stored as parquet as select * from testdb.customers;
create table employees stored as parquet as select * from testdb.employees;
create table sales stored as parquet as select * from testdb.sales;
```
Run three queries

# Run queries on Hive OCR file
```SQL
create database testdb3;
use testdb3;
create table products stored as orc as select * from testdb.products;
create table customers stored as orc as select * from testdb.customers;
create table employees stored as orc as select * from testdb.employees;
create table sales stored as orc as select * from testdb.sales;
```

Run three queries

# Prepare Big SQL environment

Configure Big SQL connection using jsqsh command line utility.
```BASH
jsqsh --setup
```
Connection wizard -> Choose bigsql connection -> Provide password (5) -> Test -> Save
When connection is configured, launch jsqsh with connection name as a parameter
```BASH
jsqsh bigsql
``` 
Before running queries, make sure that the user has EXECUTE privilege on SYSHADOOP.HCAT_SYNC_OBJECTS stored procedure.

https://www.ibm.com/support/knowledgecenter/SSCRJT_5.0.0/com.ibm.swg.im.infosphere.biginsights.db2biga.doc/doc/biga_hadsyncobj.html

# Run queries in Big SQL on Hive Parquet files.
```BASH
jsqsh bigsql
``` 
```SQL
set current schema testdb1;
values(current schema);
CALL SYSHADOOP.HCAT_SYNC_OBJECTS( 'testdb1', '.*');
\show tables;
```
Run queries

# Run queries in Big SQL on Hive OCR files.
```BASH
jsqsh bigsql
``` 
```SQL
set current schema testdb3;
values(current schema);
CALL SYSHADOOP.HCAT_SYNC_OBJECTS( 'testdb3', '.*');
\show tables;
```
Run queries

# Spark SQL

Make sure that MySQL JDBC driver is available
```BASH
ls /usr/share/java/mysql-connector-java.jar
```

Launch Spark Scala shell
```BASH
spark-shell --jars /usr/share/java/mysql-connector-java.jar --driver-memory 2g
```
Execute sequence of Scala commands. Data is loaded directly from MySQL testdb database and tranformed to Spark DF. Assign to jdbcHostname variable host name of MySQL database. show_timing function measures the execution time SQL query.
```SCALA
Class.forName("com.mysql.jdbc.Driver")
val jdbcUsername = "test"
val jdbcPassword = "test"
val jdbcHostname = {{{MySQL host name or iP address}}}
val jdbcPort = 3306
val jdbcDatabase ="testdb"
val jdbcDriver = "com.mysql.jdbc.Driver"
val jdbcUrl = s"jdbc:mysql://${jdbcHostname}:${jdbcPort}/${jdbcDatabase}?user=${jdbcUsername}&password=${jdbcPassword}"
val connProp = new java.util.Properties()
connProp.setProperty("driver", "com.mysql.jdbc.Driver") 

val salesDF = spark.read.jdbc(jdbcUrl, "SALES", connProp).cache
salesDF.show
salesDF.registerTempTable("SALES")

val customersDF = spark.read.jdbc(jdbcUrl, "CUSTOMERS", connProp).cache
customersDF.show
customersDF.registerTempTable("CUSTOMERS")

val productsDF = spark.read.jdbc(jdbcUrl, "PRODUCTS", connProp).cache
productsDF.show
productsDF.registerTempTable("PRODUCTS")

val employeeDF=spark.read.jdbc(jdbcUrl,"EMPLOYEES", connProp).cache
employeeDF.show
employeeDF.registerTempTable("EMPLOYEES")

def show_timing[T](proc: => T): T = {
    val start : Double =System.nanoTime()
    val res = proc // call the code
    val end :Double = System.nanoTime()
    val e: Double  = (end-start)/1000000000
    println("Time elapsed: " + e + " secs")
    res
}

var res = spark.sql(" select salespersonid, sum(quantity) from SALES group by salespersonid")
 show_timing({ res.show})
var res = spark.sql("select * from (select salespersonid, sum(quantity) from SALES group by salespersonid) as s,EMPLOYEES where salespersonid = employeeid")
 show_timing({ res.show})
var res = spark.sql("select * from (select c.*,q from (select customerid,sum(quantity) as q from SALES group by customerid) as s,CUSTOMERS as c where c.customerid = s.customerid) as r order by q desc LIMIT 20")
 show_timing({ res.show})
```
# Phoenix HBase SQL

Phoenix is SQL running on the top of HBase tables.

## Preparation

Make sure that at least one Phoenix server is installed and running. Unfortunately, I discovered that in HDP 2.6.2 Phoenix Query Server should be installed on every host where HBbase Region Server is running. Otherwise, HBase Region Server will not restart because of lack of some Java classes.
Also, make sure that Phoenix client files are installed on the host you want to run the test. Otherwise, log on to the machine where Phoenix Query Server is installed.

Goto HBase configuration -> Advanced -> Custom hbase-site -> Add Property -> hbase.table.sanity.check=false

Identify hbase.zookeeper.quorum configuration parameter.

Modify __hadoopsql/pho/imp__ script file. Set ZOO environment variable to {hbase.zookeeper.quorum} variable. If necessary, adjust PHOHOME variable.

Run script file
```BASH
cd hadoopsql/pho
./imp
```
File sales.txt is too big to be swallowed in one go and has to be split into several parts more digestible.

# Results

## Cluster 1, BigInsights 4.2
* 3 data node, 6 mgm nodes, HA
* mgm nodes: 8 cores 32 GB
* data nodes: 1 core 8 GB

| Engine | Query 1 | Query 2 | Query 3
|:-------|:-------:|:--------:|:------:|
| MySql | 11.57 | 30.89 | 35.17
| Hive text | 22 | 62 | 67
| Hive Parquet | 33 | 60 | 86
| Hive ORC | 27 | 63 | 79
| Big SQL Parquet | 1.66 | 1.88 | 2.26
| Big SQL ORC | 3.2 | 4.3 | 3.6
| Spark SQL | 7 | 5 | 5
| Phoenix(HBase) SQL | 17 | 19 | 17

## Cluster 2, BigInsights 4.2

* 3 mgm nodes, 2 data nodes
* all nodes: 8 cores, 64 GB

| Engine | Query 1 | Query 2 | Query 3
|:-------|:-------:|:--------:|:------:|
| MySql | 3.66 | 3.78 | 5.07
| Hive text | 20.75 | 37.028 | 42.997
| Hive Parquet | 22.148 | 40.173 | 47.962
| Hive ORC | 19.238 | 35.332 | 40.71
| Big SQL Parquet | <> | <> | <>
| Big SQL ORC | 2.987 | 0.615 | 0.996
| Spark SQL | 1 | 1 | 1-2
| Phoenix(HBase) SQL | 16.669 | 15.594 | 16.473

## Cluster 3, HDP 2.6.2
* Hive and TEZ
* joke cluster, docker, 2 nodes, mixed mgm and data
* host machine: 16 GB, 8 cores

| Engine | Query 1 | Query 2 | Query 3
|:-------|:-------:|:--------:|:------:|
| MySql | 4.45 | 4.01 | 4.19
| Hive text | 118 (159.93 + 101.211 +93.352) | 64 ( 83,569 + 78,196 + 29,612) | 140 (84,008 + 29,991 + 78,193)
| Hive Parquet | 11 ( 23,831 + 7,053 + 3,001) | 9 (17,892 + 7,289 + 2,689) | 9.6 (10,326 + 10,134 + 8,251)
| Hive ORC | 26.8 (13,902 + 61,321 + 5,286) | 1.9 ( 1.698 + 1.352 + 2.604) | 1.8 (2.690 + 1.404 + 1.381)
| Big SQL Parquet | 5.7 (14.417 + 1.405 + 1.405) | 1.8 (2.352 + 1.610 + 1.574) | 2 (2.842 + 1.571 + 1.619)
| Big SQL ORC | 4 (9.157 + 1.414 + 1.350) | 1.9 (1.698 + 1.352 + 2.604) | 1.8 (2.690 + 1.404 + 1.381)
| Spark SQL | 2.0 | 1.4 | 3,7
| Phoenix(HBase) SQL | 6.7 (12.761 + 3.515 + 3.748) | 4.1 ( 3.959 + 4.062 + 4.306) | 4.4 (3.726 + 5.328 + 4.306)

## Cluster 4, HDP 2.6.2
* Hive and TEZ
* 3 mgm nodes and 4 data nodes
* all nodes: 4 cores and 8GB

| Engine | Query 1 | Query 2 | Query 3
|:-------|:-------:|:--------:|:------:|
| MySql | 6.54 | 5.75 | 7.19
| Hive text | 11.8 (20,195 + 9,302 + 5,996) | 5.4 (10,708 + 3,037 + 2,469) | 8.8 ( 10,44 + 8,205 + 7,897)
| Hive Parquet | 5.3 (6,815 + 5,666 + 3,3) | 5.6 (6,311 + 6,655 + 3,922) | 4.5 (7,259 + 3,304 + 2,986)
| Hive ORC | 5.3 (7,138  + 4,421 + 4,561) | 6.3 (5,337 + 2,649 + 11,138) | 7.7 (8,4 + 8,349 + 6,394)
| Big SQL Parquet | 1.4 (0.805 + 1.848 + 1.613) | 1.3 (1.66 + 1.217 + 1.037) | 3.1 (6.600 + 1.467 + 1.407)
| Big SQL ORC | 4 (9.157 + 1.414 + 1.350) | 1.9 (1.698 + 1.352 + 2.604) | 1.3 (1.662 +1.092 + 1.153)
| Spark SQL | 2.0 | 1.8 | 1.7
| Phoenix(HBase) SQL | 19.1 (20,039 + 18,588 + 18,706) | 18.6 (  18,992 + 18,237 + 18,578) | 19 ( 18,602 + 19,445 + 19,065)



