# hadoopsql

Test comparing performance of different SQL engines in Hadoop/IBM BigInsights/HortonWorks HDP environment. The following SQL engines are used:
* Hive over text files
* Hive over Parquet files
* Hive over OCR files
* Big SQL over Hive Parquet files
* Big SQL over Hive OCR files
* Spark SQL
* Phoenix HBase SQL

The same data set is loaded into a particular surrounding and the same three SQL queries are launched and time spent to execute is taken.
No optimization is done to boost the execution of a particular engine. Default configuation out of the box is used.
It is not any kind of benchmarking and one should be extra careful to generalize the results. 

# Test scenario

1. Load data into MySQL database. Four tables are created: SALES, CUSTOMERS, EMPLOYEES, PRODUCTS
2. Import data from MySQL into Hive using Sqoop utility
3. Run queries over Hive text, Parquet and OCR
4. Catalog Hive tables into IBM BigSQL. Run queries using Big SQL engine.
5. Load data into Spark and execute queries using Spark SQL.
6. Load data into HBase Phoenix and run queries through Phoenix SQL

# Download data 

* git clone https://github.com/stanislawbartkowski/hadoopsql.git
* cd hadoopsql
* tar xvfz data.tgz

# Load data into MySQL database

* Prepare MySQL (MariaDB) database. Embedded HDP MySQL database can be used.

As admin user 

* CREATE DATABASE TESTDB;
* CREATE USER 'test'@'%' IDENTIFIED BY 'test';
* GRANT ALL PRIVILEGES ON *.* TO 'test'@'%';

Launch mysql console ad test user 
* cd hadoopsql
* mysql -h mod3.fyre.ibm.com -u test -p testdb

Load data

* source create.sql









