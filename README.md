# Batch processing: ETL pipeline, data modeling, warehousing and visualization of Telco customer churn (IBM dataset)

## Table of Contents
1. [Introduction](#1-introduction)
   - [Technologies used](#technologies-used)
3. [Implementation Overview](#2-implementation-overview)
4. [Design](#3-design)


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

#### Airflow system
![Untitled](https://github.com/minWang916/Batch-processing/assets/116493016/34bbeb8b-2d04-4919-9507-568b6fbf0ad5)

#### Statistical charts and graphs
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
