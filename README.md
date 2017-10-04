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
