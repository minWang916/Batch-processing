# Batch processing: ETL pipeline, data modeling, warehousing and visualization of Telco customer churn (IBM dataset)

## Table of Contents
1. [Introduction](#1-introduction)
   - [Technologies used](#technologies-used)
3. [Implementation Overview](#2-implementation-overview)
4. [Design](#3-design)
5. [Project structure](#4-project-structure)
6. [Settings](#5-settings)
   - [Prerequisites](#prerequisites)
   - [Important note : You must specify AWS credentials for each of the following](#important-note)
   - [AWS Infrastructure](#aws-infrastructure)
   - [Docker](#docker)
   - [Running](#running)
7. [Implementation](#6-implementation)
   - [Load Sales Data into PostgreSQL Database](#61-load-sales-data-into-postgresql-database)
   - [Load Data from PostgreSQL to Amazon Redshift](#62-load-data-from-postgresql-to-amazon-redshift)
8. [Visualize Result](#7-visualize-result)


## 1. Introduction 
Batch processing is a method of processing data where a group of data items is collected, processed, and executed together as a single unit. Instead of processing individual transactions in real-time, batch processing allows for the processing of multiple transactions at once, usually in a sequential manner. In this case, I use the data from Telco company customer data and work as a data engineer who collects the data batch after a period of time and analyse the data. The job consists of producing a data pipeline to gather the data from data source, store it in a datawarehouse and furher use it for analytical purpose.

<b>Source of data: </b> https://www.kaggle.com/datasets/yeanzc/telco-customer-churn-ibm-dataset?resource=download

Data includes a single csv file : <b> <i> Telco_customer_churn.csv </i> </b>

But I split it into 3 seperate files ( <b> <i> Telco_customer_churn1.csv,Telco_customer_churn2.csv,Telco_customer_churn3.csv </i> </b>) to mimic real life usage as there might be multiple data sources and the pipeline should also be able to assemble the data (AWS Glue).

### Technologies used
- Python
- Airflow
- AWS services: EC2, S3, Glue Crawler, Glue Data Catalog, Redshift
- PowerBI 

## 2. Implementation overview 
An ETL pipeline to collect raw data into actionable insights to store them in Amazon S3 for staging. Then implement another Airflow (hosted with EC2) ETL pipeline which process data from S3 and load them to Amazon Glue Crawler for analyzing data structure and schema, which are then stored in Amazon Glue Data Catalog. After that, Airflow delivers the structured data to Amazon Redshift (Data warehouse) and ultimately loaded from there to powerBI for visualization. Amazon Athena is also used for querying data directly from s3 and it is just for checking purpose. 

![System design](https://github.com/minWang916/Batch-processing/assets/116493016/e49939ec-48cd-440c-9d3a-938a690ff270)




## 3. Design 


![Untitled](https://github.com/minWang916/Batch-processing/assets/116493016/34bbeb8b-2d04-4919-9507-568b6fbf0ad5)


![graphs](https://github.com/minWang916/Batch-processing/assets/116493016/4daf2f09-a2d0-4747-8c61-ffc1d410573c)






## 4. Project Structure

```bash

Batch-Processing/
  ├── data/
  │   ├── Telco_customer_churn.csv
  │   ├── Telco_customer_churn.xlsx
  │   │── Telco_customer_churn1.csv
  │   │── Telco_customer_churn2.csv
  │   │── Telco_customer_churn3.csv
  │── customer_churn_dag.py 
  │── requirements.txt  
  │── README.md   

```
<br>

## 5. Settings

### Prerequisites
- AWS account 
- Terraform
- Docker 

### Important note

<b> You must specify AWS credentials for each of the following </b>

- S3 access : Create an IAM user "S3-admin" with <b> S3FullAccess </b> policy, create access credentials and add them to [Extract.py](/airflow/dags/ETL_psql/Extract/Extract.py) and [ETL_psql_s3.py](/airflow/dags/ETL_redshift/ETL_psql_s3.py)
- Redshift access : Create an IAM user "Redshift-admin" with <b> RedshiftFulAccess </b> policy, create accesss credentials and add them to [Load_s3_to_redshift.py](/airflow/dags/ETL_redshift/Load_s3_to_redshift.py).
- Administrator access : Create an IAM user "Admin" with <b> AdministratorAccess </b> policy, create accesss credentials and add them to [terraform.tfvars](/terraform/terraform.tfvars) and [Makefile](/Makefile). This IAM user responsible for <b> <i> provisioning Redshift cluster, and add connection between airflow and Redshift cluster </b> </i>.

### AWS Infrastructure 

<img src="/assets/Redshift%20diagram.png" alt="Redshift diagram" height="500">

- Two <b> dc2.large </b> type nodes for Redshift cluster
- Redshift cluster type : multi-node
- Redshift cluster is put inside a VPC <i> (10.10.0.0/16) </i>, redshift subnet group consists of 2 subnets <i> "Subnet for redshift az 1"(10.10.0.0/24) </i> and <i> "Subnet for redshift az 2" (10.10.1.0/24) </i>, each subnet is put in an Availability zone.

- These two subnets associate with a public route table (outbound traffic to the public internet through the Internet Gateway).
 
- Redshift security group allows all inbound traffic from port 5439. 

- Finally, IAM role is created for Redshift with full S3 Access. 

- Create redshift cluster.

```python
resource "aws_redshift_cluster" "sale_redshift_cluster" {
    cluster_identifier  = var.redshift_cluster_identifier
    database_name       = var.redshift_database_name
    master_username     = var.redshift_master_username
    master_password     = var.redshift_master_password
    node_type           = var.redshift_node_type
    cluster_type        = var.redshift_cluster_type
    number_of_nodes     = var.redshift_number_of_nodes

    iam_roles = [aws_iam_role.redshift_iam_role.arn]  

    cluster_subnet_group_name = aws_redshift_subnet_group.redshift_subnet_group.id
    skip_final_snapshot = true

    tags = {
        Name = "vupham_redshift_cluster"
    }
}
```

### Docker 
```Python
# ./docker/Dockerfile
FROM apache/airflow:2.5.1
COPY requirements.txt /
RUN pip install --no-cache-dir -r /requirements.txt 

```

[Dockerfile](/docker/Dockerfile) build a custom images with <i> apache-airflow:2.5.1 and libraries in 'requirements.txt' </i>

```python
# ./docker/requirements.txt
redshift_connector
pandas
apache-airflow-providers-amazon==8.1.0
apache-airflow-providers-postgres==5.4.0
boto3==1.26.148
psycopg2-binary==2.9.6

```
[docker-compose](/docker-compose.yaml) will build containers to run our application.

### Running 

Please refer to [Makefile](/Makefile) for more details
```
# Clone and cd into the project directory
git clone https://github.com/anhvuphamtan/Batch-Processing.git
cd Batch-Processing

# Start docker containers on your local computer
make up

# Add airflow connections : postgres-redshift-aws connections
make connections

# Set up cloud infrastructure
make infra-init # Only need in the first run 

# Notes : Please configure your own AWS access & secret keys in terraform.tfvars and /airflow/dags/ETL_redshift/Load_s3_to_redshift.py in order to create redshift cluster and connect to it
make infra-up # Build cloud infrastructure
```

## 6. Implementation

### Refer to [Implementation detail.md](/Implementation%20detail.md) for more details on implementation

### 6.1 Load sales data into PostgreSQL database

<img src=assets/ETL_psql.png alt="ETL psql" height="400">

<b> Airflow tasks </b>


 ```python
# ./dags_setup.py # Airflow dags
# -------------- Create schema task ------------- #
Create_psql_schema = PostgresOperator(
    task_id = 'Create_psql_schema',
    postgres_conn_id = 'postgres_sale_db',
    sql = 'create_pgsql_schema.sql'
)
# ---------------------------------------------- #


# ---------------- Extract task ---------------- #
Extract_from_source = PythonOperator(
    task_id = 'Extract_from_source',
    python_callable = Extract_from_source
)
# ---------------------------------------------- #


# ---------------- Transform task ---------------- #
Transform_products = PythonOperator(
    task_id = "Transform_product_df",
    python_callable = Transform_products,
    op_kwargs = {"Name" : "products", "filePath" : "products.csv"}
)

.....

Transform_shipments = PythonOperator(
    task_id = "Transform_shipment_df",
    python_callable = Transform_shipments,
    op_kwargs = {"Name" : "shipments", "filePath" : "shipments.csv"}
)
# ----------------------------------------------- #

# ----------------- Load task ----------------- #
Load_psql = PythonOperator(
    task_id = "Load_to_psql",
    python_callable = Load_schema
)
# -------------------------------------------- #
 ``` 
 
<b> 1. Create_psql_schema : </b> Create PostgreSQL schema and its tables according to our data model design.

<b> 2. Extract_from_source : </b> Extract raw data from s3 bucket and store them in <i> Input_data </i> folder.

<b> 3. Perform transformation : </b> This part split into 5 small tasks, each handle the data transformation on a specific topic.
There are 6 python files : <i> Transform.py </i>, <i> Transform_<b>name</b>.py </i> where <b> <i> name </i> </b> correspond to a topic <b> <i> ['sales', 'products', 'customers', 'shipments', 'locations']. </i> </b>
Each <i> Transform_<b>name</b>.py </i> responsible for cleaning, transforming and integrating to a corresponding OLTP table. Python class is used,
all they all inherit from the parent class in <i> Transform.py </i> :

<b> 4. Load_to_psql : </b> Load all transformed data into PostgreSQL database.

<br>

### 6.2 Load data from PostgreSQL to Amazon Redshift
<img src=assets/ETL_redshift.png alt="ETL redshift" height="400">

<b> Airflow tasks </b>
  
```python
# ./dags_setup.py # Airflow dags
ETL_s3 = PythonOperator(
    task_id = "ETL_s3",
    python_callable = ETL_s3
)

Create_redshift_schema = PythonOperator(
    task_id = "Create_redshift_schema",
    python_callable = Create_redshift_schema,
    op_kwargs = {"root_dir" : "/opt/airflow/redshift_setup"}  
)

Load_s3_redshift = PythonOperator(
    task_id = "Load_s3_redshift",
    python_callable = Load_s3_to_redshift
)
```
<br>
<b> 1. ETL_s3 : </b> Extract data from PostgreSQL database, perform transformation, and load to S3 bucket 

<b> 2. Create_redshift_schema : </b> Create redshift schema

<b> 3. Load_s3_redshift : </b> Load data from S3 bucket to Redshift
  
<br> 

## 7. Visualize result

Connect redshift to metabase and visualize results

<div style="display: flex; flex-direction: column;">
  <img src=assets/metabase.png alt="connect_metabase" height="500">
  <p style="text-align: center;"> <b> <i> Connect to metabase </i> </b> </p>
</div>

### Results

<div style="display: flex; flex-direction: column;">
  <img src=assets/Revenue%20by%20month.png alt="Revenue by month" height="500">
  <p style="text-align: center;"> <b> <i> Revenue by month in 2022 </i> </b> </p>
</div>

<br> <br>
  
<div style="display: flex; flex-direction: column;">
  <img src=assets/Brand%20popularity.png alt="Brand popularity.png" height="500">
  <p style="text-align: center;"> <b> <i> Brand popularity </i> </b> </p>
</div>

<br> <br>
  
<div style="display: flex; flex-direction: column;">
  <img src=assets/Profit%20by%20state.png alt="Profit by state.png" height="500">
  <p style="text-align: center;"> <b> <i> Profit by state </i> </b> </p>
</div>

<br> <br>

<div style="display: flex; flex-direction: column;">
  <img src=assets/Shipping%20orders%20by%20company.png alt="Shipping orders by company" height="500">
  <p style="text-align: center;"> <b> <i> Shipping orders by company </i> </b> </p>
</div>
  

