# hadoopsql

A test comparing the performance of different SQL engines in Hadoop/IBM BigInsights/HortonWorks HDP environment. The following SQL engines are used:
* Hive on text files
* Hive on Parquet files
* Hive on ORC files
* Big SQL on Hive Parquet files
* Big SQL on Hive ORC files
* Spark SQL
* Phoenix HBase SQL

The same data set is loaded into a particular surrounding and the same three SQL queries are launched and time spent to execute is taken.
No optimization is done to boost the execution of a particular engine. Default configuration out of the box is used.
It is not any kind of benchmarking and one should be extra careful to generalize the results. 

# Test scenario

1. Load data into MySQL database. Four tables are created: SALES, CUSTOMERS, EMPLOYEES, PRODUCTS
2. Import data from MySQL into Hive using Sqoop utility
3. Run queries on Hive text, Parquet and ORC
4. Catalog Hive tables into IBM BigSQL. Run queries using Big SQL engine.
5. Load data into Spark and execute queries using Spark SQL.
6. Load data into HBase Phoenix and run queries through Phoenix SQL
* There are huge differences in execution time while running the same command so I used to run the query three times one after the other and calculate the arithmetic mean.

# Queries under test

Queries are simple: aggregate over a single table, join two tables and join nested table.

1. select salespersonid, sum(quantity) from SALES group by salespersonid
2. select * from (select salespersonid, sum(quantity) from SALES group by salespersonid) as s,EMPLOYEES where salespersonid = employeeid
3. select * from (select c.*,q from (select customerid,sum(quantity) as q from SALES group by customerid) as s,CUSTOMERS as c where c.customerid = s.customerid) as r order by q desc LIMIT 20
9. SELECT C.FIRSTNAME,C.LASTNAME,V.* FROM (SELECT P.PRODUCTID,S.SALESID,C.CUSTOMERID AS CID, S.QUANTITY * P.PRICE AS VAL FROM SALES S,PRODUCTS P,CUSTOMERS C WHERE P.PRODUCTID=S.PRODUCTID AND S.CUSTOMERID = C.CUSTOMERID GROUP BY P.PRODUCTID,S.SALESID,C.CUSTOMERID,S.QUANTITY,P.PRICE) AS V, CUSTOMERS AS C WHERE V.CID =  C.CUSTOMERID ORDER BY V.VAL DESC LIMIT 20
9. SELECT /*+ USE_SORT_MERGE_JOIN */ C.FIRSTNAME,C.LASTNAME,V.* FROM (SELECT P.PRODUCTID,S.SALESID,C.CUSTOMERID AS CID, S.QUANTITY * P.PRICE AS VAL FROM SALES S,PRODUCTS P,CUSTOMERS C WHERE P.PRODUCTID=S.PRODUCTID AND S.CUSTOMERID = C.CUSTOMERID GROUP BY P.PRODUCTID,S.SALESID,C.CUSTOMERID,S.QUANTITY,P.PRICE) AS V, CUSTOMERS AS C WHERE V.CID =  C.CUSTOMERID ORDER BY V.VAL DESC LIMIT 20

The last one should be executed on Phoenix (HBase). It is the same as the previous one with the hint inside

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
CREATE DATABASE testdb;
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

* Customize imphive script file, provide host name for MySQL database. Sometimes it is necessary to use capital letter; instead of testdb, TESTDB
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
beeline -u "jdbc:hive2://{hive server}:10000/{databasename}" -n {username} $@
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
Run four queries

# Run queries on Hive ORC file
```SQL
create database testdb3;
use testdb3;
create table products stored as orc as select * from testdb.products;
create table customers stored as orc as select * from testdb.customers;
create table employees stored as orc as select * from testdb.employees;
create table sales stored as orc as select * from testdb.sales;
```

Run four queries

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

# Run queries in Big SQL on Hive ORC files.
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

The code below is relevant for Spark 2.X. To execute this code in Spark 1.6 replace spark with sqlContext. For instance:
```SCALA
val employeeDF=sqlContext.read.jdbc(jdbcUrl,"EMPLOYEES", connProp).cache
```

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

Launch phoenix command line
```BASH
phoenix-sqlline {zookeeper quorum}:/hbase-unsecure
```
Run queries

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

## Cluster 3, HDP 2.6.3
* Hive and TEZ
* joke cluster, docker, 2 nodes, mixed mgm and data
* host machine: 16 GB, 8 cores

| Engine | Query 1 | Query 2 | Query 3 
|:-------|:-------:|:--------:|:------:|
| MySql | 4.45 | 4.01 | 4.19 
| Hive text | 118 (159.93 + 101.211 +93.352) | 64 ( 83,569 + 78,196 + 29,612) | 140 (84,008 + 29,991 + 78,193) |
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
| Hive text | 11.8 (20,195 + 9,302 + 5,996) | 5.4 (10,708 + 3,037 + 2,469) | 8.8 ( 10,44 + 8,205 + 7,897) |
| Hive Parquet | 5.3 (6,815 + 5,666 + 3,3) | 5.6 (6,311 + 6,655 + 3,922) | 4.5 (7,259 + 3,304 + 2,986)
| Hive ORC | 5.3 (7,138  + 4,421 + 4,561) | 6.3 (5,337 + 2,649 + 11,138) | 7.7 (8,4 + 8,349 + 6,394)
| Big SQL Parquet | 1.4 (0.805 + 1.848 + 1.613) | 1.3 (1.66 + 1.217 + 1.037) | 3.1 (6.600 + 1.467 + 1.407)
| Big SQL ORC | 4 (9.157 + 1.414 + 1.350) | 1.9 (1.698 + 1.352 + 2.604) | 1.3 (1.662 +1.092 + 1.153)
| Spark SQL | 2.0 | 1.8 | 1.7
| Phoenix(HBase) SQL | 19.1 (20,039 + 18,588 + 18,706) | 18.6 (  18,992 + 18,237 + 18,578) | 19 ( 18,602 + 19,445 + 19,065)

## Cluster 5, BigInsights 4.2.0
* No TEZ, no Big SQL
* 5 mgm nodes and 8 data nodes
* all nodes: 20 cores and 32GB

| Engine | Query 1 | Query 2 | Query 3
|:-------|:-------:|:--------:|:------:|
| MySql | (4.22 + 4.02 + 3.82) | (3.81 + 5.81 + 4.19) | (4.18 + 4.23 + 4.27)
| Hive text | (20.269 + 18.578 + 18.689) | (39.629 + 36.035 + 39.466) |  ( 42.713 + 45.542 + 49.328)
| Hive Parquet | (21.507 + 21.551 + 21.542) | (46.15 + 40.978 + 39.897) | (46.317 + 46.298 + 51.62)
| Hive ORC | (21.075 + 18.799 + 23.146) | (35.913 + 39.015 + 37.777) | (44.723 + 43.983 + 50.171)
| Big SQL Parquet | | | 
| Big SQL ORC | | | 
| Spark SQL | (1.32 + 0.98 + 0.89) | (0.93 + 0.80 + 0.76) | (1.00 + 0.88 + 0.83)
| Phoenix(HBase) SQL | (8.753 + 9.964 + 9.573) | (9.378 + 8.67 + 9.761) | (8.978 + 8.889 + 9.054)

## Cluster 6, HDP 2.6.3
* PowerPC
* 2 mgm nodes and 2 data nodes
* all nodes: 8 cores and 64GB

| Engine | Query 1 | Query 2 | Query 3 | Query 4
|:-------|:-------:|:--------:|:------:|:------:|
| MySql | 11.25 (11.23 + 11.22 + 11.31) | 11.21 (11.18 + 11.24 + 11.2) | 11.79 (11.58 + 11.92 + 11.87) | (unable to execute)
| Hive text | 11.31 (19.373 + 8.812 + 5.761) | 7.28 (8.068 + 7.598 + 6.167) | 8.55 (9.528 + 8.101 + 8.006) | 42.06 ( 45.8 + 39.085 + 41.306)
| Hive Parquet | 14.28 (7.802 +  3.392 + 3.092) | 3.22 (3.582 + 2.905 + 3.179) | 21.13 (8.42 + 7.12 + 5.591) | 46.45 44.506 + 47.493 + 47.344)
| Hive ORC | 3.07 (6.312 + 1.608 + 1.281) | 3.50 (5.797 + 2.171 + 2.539) | 4.28 (7.697 + 2.67 + 2.477) | 35.74 (33.779 + 33.042 + 40.385)
| Big SQL Parquet | | | 
| Big SQL ORC | | | 
| Spark SQL | 1.47 ( 2.3918 + 1.033 + 0.995) | 2.20 (1.557 + 1.0159 + 1.042) | 1.28 (1.617 + 1.156 + 1.076) | Unable to execute
| Phoenix (HBase) | 21.76 (22.833 + 21.223 +21.23) | 21.05 (21.142 + 21.123 + 20.873) | 21.45 (21.615 + 21.383 + 21.34) | 96.61 (98.762 + 94.461)

## Cluster 7, HDP 2.6.3 + Big SQL 5.0.1
* TEZ
* 6 mgm nodes + 2 data nodes 
* mgm nodes : 256GB, 15 cores
* data nodes: 1TB, 72 corres

| Engine | Query 1 | Query 2 | Query 3 | Query 4
|:-------|:-------:|:--------:|:------:|:------:|
| MySQL | 3.95 (3.39 + 4.00 + 4.45) | 4.02 (3.91 + 4.22 + 3.94) | 4.64 ( 5.30 + 4.39 + 4.22) | unable to execute
| Hive text | 9.34 (12.716 + 8.203 + 7.09) | 7.44 ( 8.45 + 7.199 + 6.675) | 10.03 (10.277 + 9.727 + 10.072) | 23.43 (24.184 + 23.906 + 22.206)
| Hive Parquet | 7.66 (9.958+8.201+4.832) | 4.69 (4.409+ 4.479 + 5.184) | 8.27 (9.95 + 6.807 + 8.047) | 19.65 (20.976+19.508 +18.457)
| Hive ORC | 5.11 (6.518 + 3.864 + 4.949) | 5.18 (7.085 + 4.064 + 4.378) | 6.81 (8.476 + 5.582 + 6.371) | 21.61 (23.74 + 22.611 + 18.472)
| Big SQL Parquet | 0.329 (0.390 + 0.298 + 0.299) | 0.46 (0.508 + 0.431 + 0.435) | 1.421 (1.401 + 1.444 + 1.418) | 44.60 (75.255 + 55.309 + 3.247 )
| Big SQL ORC | 2.07 (5.740 + 0.243 + 0.231) | 0.30 (0.349 + 0.280 + 0.269) | 0.86 (0.923 + 0.825 + 0.844) | 58.05 (58.529 + 57.853 + 57.762)
| Spark SQL | 0.76 (1.03 + 0.62 + 0.64) | 0.63 ( 0.77 + 0.62 + 0.51) | 0.59 (0.71 + 0.54 + 0.51) | 9.65 (10.36 + 9.45 + 9.15 )
| Phoenix (HBase) | 11.44 (11.714 + 11.275 + 11.318) | 11.16 (11.356 + 10.856 + 11.264) | 11.04 (11.297 + 10.888 + 10.928) | 54.58 (55.04 + 54.112 + 54.6) 

## Cluster 8, HDP 2.6.3 + Big SQL 5.0.1
* TEZ
* 6 mgm nodes + 7 data nodes 
* mgm nodes : 256GB, 15 cores
* data nodes: 1TB, 72 corres

| Engine | Query 1 | Query 2 | Query 3 | Query 4
|:-------|:-------:|:--------:|:------:|:------:|
| MySQL |  |  | | unable to execute
| Hive text |7.086(7.915+6.029+7.314) | 6.832(7.157+7.217+6.122) |11.439 (13.718+9.824+10.776)  | 25.216(26.549+25.189+23.909)
| Hive Parquet | 10.535 (17.286+8.666+5.652)| 6.811(7.721+6.324+6.387) | 9.265(11.184 + 9.704+6.908) | 23.36(25.19 + 21.587+23.303)
| Hive ORC | 5.699(6.916+4.272+5.908) | 5.51(4,972+6.618+4.94)  | 10.097 (8.809+12.202+9.28) | 20.554(20.346+21.281+20.034)
| Big SQL Parquet | 0.579(0.829+0.449+0.459) | 0.235(0.271+0.218+0.216)| 0.386(0.408+0.360+0.391)| 3.713(1.777 + 1.450+1.459)
| Big SQL ORC | 0.548(0.761+0.448+0.436) | 0.445(0.470+0.427+0.437) | 0.288(0.431+0.209+0.225) | 1.479(1.741+1.348+1.349)
| Spark SQL | 0.41(0.433+0.403+0.395) | 0.479(0.468 + 0.483 + 0.486)  | 0.547(0.668 + 0.479 + 0.494) |9.051(8.978+8.948+9.227) 
| Phoenix (HBase) | 2.909(3.369+2.763+2.594) | 0.479(0.468 + 0.483 + 0.486) | 0.547(0.668 + 0.479 + 0.494) | 9.051(8.978 + 8.948 + 9.227)




