# hadoopsql

Test comparing performance of different SQL engines in Hadoop/IBM BigInsights/HortonWorks HDP environment. The following SQL engines are used:
* Hive on text files
* Hive on Parquet files
* Hive on OCR files
* Big SQL on Hive Parquet files
* Big SQL on Hive OCR files
* Spark SQL
* Phoenix HBase SQL

The same data set is loaded into a particular surrounding and the same three SQL queries are launched and time spent to execute is taken.
No optimization is done to boost the execution of a particular engine. Default configuation out of the box is used.
It is not any kind of benchmarking and one should be extra careful to generalize the results. 

# Test scenario

1. Load data into MySQL database. Four tables are created: SALES, CUSTOMERS, EMPLOYEES, PRODUCTS
2. Import data from MySQL into Hive using Sqoop utility
3. Run queries on Hive text, Parquet and OCR
4. Catalog Hive tables into IBM BigSQL. Run queries using Big SQL engine.
5. Load data into Spark and execute queries using Spark SQL.
6. Load data into HBase Phoenix and run queries through Phoenix SQL
* There are huge differences in execution time while running the same command so I used to run the query three times one after the other and take the mean.

# Queries under test

Queries are simple : aggregate over single table, join two tables and join three tables.

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
mysql -h mod3.fyre.ibm.com -u test -p testdb
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
Execute sequence of Scala commands. Data is loaded directly from MySQL testdb database and tranformed to Spark DF. Assign to jdbcHostname variable host name of MySQL database. show_timing function measure the execution time SQL query.
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
Also make sure that Phoenix client file are installed on the host you want to run the test. Otherwise log on to the machine where Phoenix Query Server is installed.

Goto HBase configuration -> Advanced -> Custom hbase-site -> Add Property -> hbase.table.sanity.check=false

Identify hbase.zookeeper.quorum configuration parameter.

Modify __hadoopsql/pho/imp__ script file. Set ZOO environment variable to {hbase.zookeeper.quorum} variable. If necessary, adjust PHOHOME variable.

Run script file
```BASH
cd hadoopsql/pho
./imp
```
File sales.txt is to big to be swallowed in one go and has to be split to several parts more digestible.




