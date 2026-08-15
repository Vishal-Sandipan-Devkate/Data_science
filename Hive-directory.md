# Apache Hive with MySQL Metastore on Hadoop: Installation, Configuration, and Usage

## Introduction

**Apache Hive** is a data warehouse and SQL-based query system built on top of Hadoop. It allows users to work with large datasets stored in **HDFS** using a SQL-like language called **HiveQL**, without having to write complex Java MapReduce programs for every analysis task.

Hive is particularly useful for users who are familiar with SQL but need to analyze data stored in a distributed Hadoop environment.

A typical Hive architecture looks like this:

```text
                    User
                     |
                     v
                 HiveQL / SQL
                     |
                     v
                Apache Hive
                     |
          +----------+----------+
          |                     |
          v                     v
      Metastore                Hadoop
          |                     |
        MySQL                   HDFS
                                |
                                v
                         Data Processing
                         MapReduce/YARN
```

In this setup:

* **Hive** provides the SQL interface.
* **HDFS** stores the actual data.
* **MySQL** stores Hive metadata.
* **YARN** manages cluster resources.
* **MapReduce** can execute Hive queries in Hadoop-based configurations.

---

# 1. Why Use MySQL as the Hive Metastore?

Hive needs a **Metastore** to store information about databases, tables, columns, partitions, locations, and other metadata.

For example, when we create:

```sql
CREATE TABLE employees (
    id INT,
    name STRING,
    department STRING,
    salary FLOAT
);
```

Hive needs to remember information such as:

```text
Database       → analytics
Table          → employees
Column 1       → id / INT
Column 2       → name / STRING
Column 3       → department / STRING
Column 4       → salary / FLOAT
Location       → HDFS warehouse path
```

This information is called **metadata**.

For development purposes, Hive can use Apache Derby, but Derby is primarily suitable for simple or single-user setups. Using **MySQL as an external Metastore database** provides a more persistent and practical architecture.

The architecture becomes:

```text
                   Hive
                    |
                    |
             +------+------+
             |             |
             v             v
          MySQL           HDFS
       Metastore         Data Files
```

---

# 2. Installing MySQL

First, update the Ubuntu package repositories:

```bash
sudo apt update
```

Install MySQL Server:

```bash
sudo apt install mysql-server -y
```

MySQL will be used to store Hive's metadata.

---

# 3. Installing the MySQL JDBC Driver

Hive needs a JDBC driver to communicate with MySQL.

Download the MySQL Connector/J driver:

```bash
wget https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.3.0/mysql-connector-j-8.3.0.jar
```

The downloaded JAR file will later be placed inside Hive's `lib` directory.

---

# 4. Creating the Hive Metastore Database

Log in to MySQL:

```bash
sudo mysql
```

Create a database for Hive:

```sql
CREATE DATABASE metastore_db;
```

Create a dedicated Hive user:

```sql
CREATE USER 'hiveuser'@'localhost'
IDENTIFIED BY 'hivepassword';
```

Grant the user access to the database:

```sql
GRANT ALL PRIVILEGES ON metastore_db.*
TO 'hiveuser'@'localhost';
```

Apply the privilege changes:

```sql
FLUSH PRIVILEGES;
```

Exit MySQL:

```sql
EXIT;
```

The resulting architecture is:

```text
              Apache Hive
                   |
                   | JDBC
                   v
            +-------------+
            |    MySQL    |
            |             |
            | metastore_db|
            +-------------+
```

---

# 5. Installing Apache Hive

For this setup, Hive **3.1.3** is used.

Download Hive:

```bash
wget https://archive.apache.org/dist/hive/hive-3.1.3/apache-hive-3.1.3-bin.tar.gz
```

Extract the archive:

```bash
tar -xvzf apache-hive-3.1.3-bin.tar.gz
```

Move Hive to `/usr/local/hive`:

```bash
sudo mv apache-hive-3.1.3-bin /usr/local/hive
```

Change ownership:

```bash
sudo chown -R $USER:$USER /usr/local/hive
```

Copy the MySQL JDBC driver into Hive's library directory:

```bash
cp mysql-connector-j-8.3.0.jar /usr/local/hive/lib/
```

---

# 6. Configure Hive Environment Variables

Open the `.bashrc` file:

```bash
nano ~/.bashrc
```

Add:

```bash
export HIVE_HOME=/usr/local/hive
export PATH=$PATH:$HIVE_HOME/bin
```

Save the file and reload the configuration:

```bash
source ~/.bashrc
```

Verify Hive:

```bash
hive --version
```

You should see information about the installed Hive version.

---

# 7. Configuring `hive-site.xml`

Hive's main configuration file is:

```text
/usr/local/hive/conf/hive-site.xml
```

Open it:

```bash
nano /usr/local/hive/conf/hive-site.xml
```

Configure the MySQL connection:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>

    <!-- MySQL JDBC Connection -->
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>
            jdbc:mysql://localhost:3306/metastore_db?createDatabaseIfNotExist=true&amp;useSSL=false
        </value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.cj.jdbc.Driver</value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>hiveuser</value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>hivepassword</value>
    </property>

    <!-- HDFS Warehouse -->
    <property>
        <name>hive.metastore.warehouse.dir</name>
        <value>/user/hive/warehouse</value>
    </property>

    <property>
        <name>hive.metastore.schema.verification</name>
        <value>false</value>
    </property>

</configuration>
```

This configuration tells Hive:

* Which MySQL database to use
* Which JDBC driver to use
* Which MySQL username and password to use
* Where Hive should store managed-table data in HDFS

---

# 8. Understanding the Hive Warehouse

Hive uses a warehouse directory in HDFS to store data belonging to managed tables.

In this setup:

```text
/user/hive/warehouse
```

is the default warehouse location.

The architecture is:

```text
                  Hive
                   |
           +-------+-------+
           |               |
           v               v
        MySQL             HDFS
      Metadata          Warehouse
                           |
                           v
                    Table Data Files
```

The important distinction is:

> **MySQL stores metadata, while HDFS stores the actual table data.**

---

# 9. Fixing the Guava Library Compatibility

Hive 3.1.3 and Hadoop 3.4.0 can contain different versions of the **Guava** library.

A library-version mismatch can result in runtime errors such as:

```text
NoSuchMethodError
```

One way to align the libraries is to remove Hive's older Guava JAR and copy the compatible Hadoop version.

Remove the older Hive Guava library:

```bash
rm /usr/local/hive/lib/guava-19.0.jar
```

Copy the Hadoop Guava library:

```bash
cp $HADOOP_HOME/share/hadoop/common/lib/guava-*.jar /usr/local/hive/lib/
```

This helps prevent incompatibility between Hive and Hadoop dependencies.

> **Note:** Dependency compatibility can vary depending on the exact Hadoop, Hive, Java, and connector versions. If Hive reports a different dependency conflict, check the specific JAR versions involved rather than blindly removing libraries.

---

# 10. Start Hadoop Services

Hive depends on Hadoop services such as HDFS and, for distributed execution, YARN.

Start HDFS:

```bash
start-dfs.sh
```

Start YARN:

```bash
start-yarn.sh
```

Verify the daemons:

```bash
jps
```

You should see processes such as:

```text
NameNode
DataNode
SecondaryNameNode
ResourceManager
NodeManager
Jps
```

---

# 11. Create Hive Directories in HDFS

Create the Hive warehouse directory:

```bash
hdfs dfs -mkdir -p /user/hive/warehouse
```

Create the temporary directory:

```bash
hdfs dfs -mkdir -p /tmp
```

Give group-write permissions:

```bash
hdfs dfs -chmod g+w /user/hive/warehouse
hdfs dfs -chmod g+w /tmp
```

These directories allow Hive to store and process data in HDFS.

---

# 12. Initialize the Hive Metastore

Before using Hive with MySQL, the required Metastore tables must be created in the MySQL database.

Run:

```bash
schematool -dbType mysql -initSchema
```

The schema tool creates the internal tables required by Hive.

Conceptually:

```text
             schematool
                  |
                  v
          MySQL metastore_db
                  |
        +---------+---------+
        |         |         |
      Tables   Columns   Metadata
```

If the initialization completes successfully, you should see a successful completion message in the terminal.

---

# 13. Starting the Hive Shell

Start the Hive command-line interface:

```bash
hive
```

You should see a prompt similar to:

```text
hive>
```

You can now execute HiveQL commands.

---

# 14. Creating a Database

Create an analytics database:

```sql
CREATE DATABASE analytics;
```

Select the database:

```sql
USE analytics;
```

Hive now operates within the `analytics` database.

---

# 15. Creating a Hive Table

Create an employee table:

```sql
CREATE TABLE employees (
    id INT,
    name STRING,
    department STRING,
    salary FLOAT
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ',';
```

The table contains four columns:

| Column       | Type   | Description         |
| ------------ | ------ | ------------------- |
| `id`         | INT    | Employee ID         |
| `name`       | STRING | Employee name       |
| `department` | STRING | Employee department |
| `salary`     | FLOAT  | Employee salary     |

Hive stores the table metadata in MySQL while the actual table data is stored in HDFS.

---

# 16. Inserting Data

Insert sample records:

```sql
INSERT INTO employees VALUES
(1, 'Alice', 'Engineering', 85000),
(2, 'Bob', 'Data Science', 92000),
(3, 'Charlie', 'Engineering', 78000);
```

Depending on the execution engine and Hive configuration, an `INSERT` operation may launch a distributed execution job.

The conceptual workflow is:

```text
HiveQL
   |
   v
Hive Compiler
   |
   v
Execution Plan
   |
   v
YARN
   |
   v
Processing Engine
   |
   v
HDFS
```

---

# 17. Querying Data Using HiveQL

You can now execute SQL-like queries.

For example:

```sql
SELECT *
FROM employees;
```

To calculate the average salary by department:

```sql
SELECT department, AVG(salary)
FROM employees
GROUP BY department;
```

The expected result is approximately:

```text
Data Science    92000
Engineering     81500
```

The Engineering average is:

```text
(85000 + 78000) / 2 = 81500
```

---

# 18. Managed Tables vs External Tables

One of the most important concepts in Hive is the difference between **managed tables** and **external tables**.

## Managed/Internal Table

A normal table created using:

```sql
CREATE TABLE employees (...);
```

is a managed table unless otherwise specified.

Hive manages:

* Table metadata
* Table data

For example:

```sql
DROP TABLE employees;
```

can remove both the table metadata and its associated managed-table data.

---

## External Table

An external table tells Hive that the underlying data is managed outside the Hive table lifecycle.

Example:

```sql
CREATE EXTERNAL TABLE employees_external (
    id INT,
    name STRING,
    department STRING,
    salary FLOAT
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
LOCATION '/data/employees';
```

In this case:

```text
MySQL
  |
  | Metadata
  v
Hive External Table
  |
  | Points to
  v
HDFS /data/employees
```

If the external table is dropped:

```sql
DROP TABLE employees_external;
```

the Hive metadata is removed, but the underlying files in the specified HDFS location are normally preserved.

---

# 19. Managed vs External Tables

| Feature                   | Managed Table   | External Table                   |
| ------------------------- | --------------- | -------------------------------- |
| Metadata                  | Hive manages it | Hive manages it                  |
| Data                      | Hive manages it | Data location managed separately |
| HDFS data on `DROP TABLE` | Usually deleted | Normally preserved               |
| Recommended for           | Hive-owned data | Existing/shared datasets         |
| `LOCATION` required       | No              | Usually specified                |

---

# 20. Complete Hive Architecture

The complete setup can be visualized as:

```text
                         User
                           |
                           v
                       HiveQL
                           |
                           v
                    Apache Hive
                           |
              +------------+------------+
              |                         |
              v                         v
         Hive Metastore              HDFS
              |                         |
              v                         v
            MySQL                 Warehouse/Data
              |                         |
              +------------+------------+
                           |
                           v
                          YARN
                           |
                           v
                  Processing Engine
```

Each component has a specific responsibility:

```text
Hive       → SQL interface and query planning
MySQL      → Stores Hive metadata
HDFS       → Stores actual data
YARN       → Manages cluster resources
MapReduce  → Distributed processing engine
```

---

# 21. Real-World Use of Hive

Hive is useful when organizations need to analyze very large datasets using SQL-like queries.

Typical applications include:

* Log analysis
* Customer analytics
* Sales analysis
* Website analytics
* Data warehousing
* ETL processing
* Reporting
* Business intelligence
* Historical data analysis

For example, a company could store millions of sales records in HDFS and use Hive to calculate:

```sql
SELECT product_id, SUM(quantity)
FROM sales
GROUP BY product_id;
```

Instead of manually writing a MapReduce program, analysts can express the operation using HiveQL.

---

# 22. Hive vs Traditional SQL Database

Hive and traditional relational databases both support SQL-like queries, but their primary purposes differ.

| Feature          | Traditional RDBMS      | Hive                           |
| ---------------- | ---------------------- | ------------------------------ |
| Primary purpose  | Transaction processing | Large-scale analytics          |
| Data size        | Usually smaller        | Very large datasets            |
| Storage          | Database storage       | HDFS/data lake storage         |
| Query language   | SQL                    | HiveQL                         |
| Transactions     | Strong OLTP support    | Primarily analytical workloads |
| Processing       | Database engine        | Distributed processing         |
| Typical workload | OLTP                   | OLAP                           |

Hive is therefore better understood as a **data warehouse and analytical query system**, rather than a direct replacement for transactional databases such as MySQL.

---

# 23. Important Commands Summary

### Check Hive Version

```bash
hive --version
```

### Start Hadoop

```bash
start-dfs.sh
start-yarn.sh
```

### Check Hadoop Processes

```bash
jps
```

### Create Hive Warehouse

```bash
hdfs dfs -mkdir -p /user/hive/warehouse
```

### Initialize Metastore

```bash
schematool -dbType mysql -initSchema
```

### Start Hive

```bash
hive
```

### Check HDFS Warehouse

```bash
hdfs dfs -ls /user/hive/warehouse
```

---

# 24. Overall Workflow

The complete installation and execution process can be summarized as:

```text
        Ubuntu
           |
           v
       Hadoop 3.4.0
           |
     +-----+-----+
     |           |
    HDFS        YARN
     |           |
     +-----+-----+
           |
           v
        Apache Hive
           |
     +-----+-----+
     |           |
   MySQL        HDFS
 Metastore    Data Storage
     |           |
     +-----+-----+
           |
           v
         HiveQL
           |
           v
      Query Execution
           |
           v
        Results
```

---

# Conclusion

**Apache Hive** provides a convenient SQL-based interface for analyzing large datasets stored in Hadoop.

In this setup, **MySQL acts as the persistent Hive Metastore**, storing information about databases, tables, columns, and other metadata. **HDFS stores the actual data**, while **YARN manages the resources required for distributed processing**.

The combination can be summarized as:

```text
MySQL  → Metadata
HDFS   → Data
YARN   → Resources
Hive   → SQL Analytics
```

This architecture demonstrates how traditional SQL concepts can be combined with Hadoop's distributed storage and processing capabilities, making large-scale data analysis easier for users familiar with SQL.
