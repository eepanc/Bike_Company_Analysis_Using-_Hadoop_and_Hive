# Bike-Company-Analysis-using-Hadoop-and-Hive

<img width="582" alt="Cloud Architecture" src="">

In this project, we will be using a suite of Apache Hadoop tools hosted in GCP Dataproc for the creation of a relational database and analysis of a bike company dataset.

*Apache Hadoop is an open-source framework for distributed storage and processing of large datasets. It is designed to handle massive amounts of data in a distributed and fault-tolerant manner. Hadoop is widely used for big data processing and analytics in a variety of industries.*

The following Hadoop tools and others were used:

- **Google Cloud Platform (GCP)**: The cloud chosen for this project. I chose this cloud because of Dataproc, this way I don't have to worry about the admin overhead that deploying an on-premises Hadoop infrastructure would give me.
- **Hive**: Used to create a data warehouse environment inside Hadoop, we will insert all our tables from HDFS into Hive to query the data for EDA (Exploratory Data Analysis).
- **DBeaver**: It's an open-source database management tool that provides a graphical interface for working with various database management systems (DBMS), including MySQL Workbench. I prefer this instead of MySQL Workbench because the GUI is more intuitive and I can use it with other database management systems, like PostgreSQL, SQL Server, AWS RDS, etc.

## Deploying the infrastructure

Once the dataset has been cleaned locally using using Pandas, I managed to ingest the data into a MySQL database using DBeaver. This is the ERD created for this dataset:

<img width="892" alt="Data Model" src="">

Now we will deploy the infrastructure on the cloud. Go to GCP and create a new Dataproc cluster, there is no need to create a robust cluster because we will be using a small dataset. The cluster already has Hive installed but we have to install Sqoop manually. 

### Using Cloud Storage FUSE

GCSFUSE is an application that allows you to mount a GCP 'bucket' on your VM's file system. By doing this, when you upload files to the bucket you can use them inside the VM. To do that, we connect to the VM via SSH and run the following command:

```
export GCSFUSE_REPO=gcsfuse-`lsb_release -c -s`
echo "deb http://packages.cloud.google.com/apt $GCSFUSE_REPO main" | sudo tee /etc/apt/sources.list.d/gcsfuse.list
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo apt-get update
sudo apt-get install gcsfuse
```

Now we can upload our tables inside a Cloud Storage bucket and transfer them to our VM:

```
mkdir <folder_name>
gcsfuse <bucket_name> <folder_name>
```

We have the files inside our VM. Now we can send them to Hive and query some data there using SQL.

### Uploading files to Hive

We will use Beeline instead of Hive, Beeline is an enhanced replacement for the Hive CLI. To log into Beeline, we type the command down below:

```
beeline -u jdbc:hive2://localhost:10000/default -n <user>@<cluster> -d org.apache.hive.jdbc.HiveDriver
```

- **-u jdbc:hive2://localhost:10000/default**: This is the JDBC URL that specifies the connection details. Here's what each part means:
    - jdbc:hive2://: This is the protocol for connecting to Hive using JDBC.
    - localhost: Replace this with the hostname or IP address of your Hive server. It currently points to "localhost," which means you're trying to connect to a Hive server running on your local machine.
    - 10000: This is the default port number for HiveServer2. Make sure it matches the port on which your Hive server is running.
    - default: This is the name of the Hive database you want to connect to. You can replace it with the name of your target database.
- **-n <user>@<cluster>**: This part specifies the username and cluster information. Replace <user> with your Hive username, and <cluster> with the name of your Hive cluster or instance.
- **-d org.apache.hive.jdbc.HiveDriver**: This part specifies the JDBC driver class to use, which is the correct driver for Hive.

We now create a database inside and check for its existence:

```
CREATE DATABASE IF NOT EXISTS <your_database_name>;
SHOW DATABASES;
```



Instead of just creating a simple database, we can also create it with some interesting metadata, and properties and even change the location in HDFS, like in the command below:

```
CREATE DATABASE IF NOT EXISTS <your_database_name>
COMMENT "<insert_your_comment_here>"
LOCATION "/user/<example>"
with DBPROPERTIES('Date' = 'xxxx-xx-xx', 'Country' = 'xx', 'Creator' = '<name>');
```

You can check your recent database created with "DESCRIBE DATABASE EXTENDED <your_database_name>;".

```
CREATE DATABASE IF NOT EXISTS test_db
COMMENT "The best database"
LOCATION "/user/example1"
with DBPROPERTIES('Date' = '2023-12-01', 'Country' = 'ND', 'Creator' = 'Horace MacArthur');
```



Now we have to put the tables from our local directory to HDFS. To do that, we first have to create a parent directory to start creating folders in HDFS, then we create a folder inside HDFS and transfer the files in there:

```
hdfs dfs -mkdir -p hdfs://<cluster>/user/<user> # create the parent directory
hadoop fs -mkdir bike_company # create a folder inside HDFS
hdfs dfs -put . bike_company # transfer all the csv files in your current local folder into the folder we just created
```

Go back to Beeline. We have to create an SQL script to create the schema for each table and another SQL script to insert all the tables in HDFS to Hive. You can find these SQL DDL statements in [this folder](). After running all the SQL scripts, we can see the tables inside Hive by using "SHOW TABLES":


## EDA

Now that we have a relational database system in Hive, we can start making some Exploratory Data Analysis (EDA) on our data:

1. Find the different types of "PersonType" in the "Person" table.

```
SELECT DISTINCT PersonType
FROM Person;
```



2. Find the top 10 biggest orders done so far.

```
SELECT totaldue
FROM salesOrderHeader
ORDER BY totaldue DESC
LIMIT 10;
```



3. Write a query that returns the number of rows in the Sales.SalesOrderDetail table by the SalesOrderID field, provided they have at least three rows of details.

```
SELECT SalesOrderID as id, COUNT(*) AS qtd 
FROM salesOrderDetail as sod
GROUP BY SalesOrderID
HAVING qtd >= 3
ORDER BY qtd DESC
LIMIT 10;
```


4. Write a query linking the Person.Person, Sales.Customer, and Sales.SalesOrderHeader tables to get a list of customer names and a count of orders placed.

```
SELECT c.CustomerID AS id, CONCAT(p.FirstName, ' ', p.LastName) AS nome, COUNT(*) AS qtd
FROM salesOrderHeader soh
JOIN customer c ON soh.CustomerID = c.CustomerID
JOIN person p ON c.PersonID = p.BusinessEntityID 
GROUP BY c.CustomerID, p.FirstName, p.LastName
ORDER BY qtd DESC
LIMIT 10;
```



5. Write a query showing the SalesOrderID, OrderDate, and TotalDue fields from the Sales.SalesOrderHeader table. Get only the lines where the order was placed during September/2011 and the total due is above 1,000. Sort by descending total due.

```
SELECT 
SalesOrderID as id,
CAST(OrderDate AS DATE) AS data, 
TotalDue AS total_devido
FROM salesOrderHeader
WHERE OrderDate BETWEEN '2011-09-01' AND '2011-09-30' AND TotalDue > 1000
ORDER BY total_devido;
```





## Conclusions

This dataset was too small for Hadoop to work properly, a simple Pandas query locally would be much faster compared to all the maps and reduces that Hadoop has to do in every single query. Hadoop is a better choice for handling large datasets when you need distributed processing, fault tolerance, scalability, and an ecosystem of tools for big data analytics. Pandas, on the other hand, is more appropriate for smaller datasets that can fit into memory on a single machine and for tasks that don't require distributed computing capabilities.

There were plenty of queries that worked in a MySQL database but did not work in Hive. I had to adapt a lot of the queries for it to work on Hive.
