# Download the dataset 

- Visit the URL https://www.kaggle.com/datasets/mkechinov/ecommerce-behavior-data-from-multi-category-store
- Download the dataset for `2019-Oct.csv.zip` and `2019-Nov.csv.zip` 
- Save the dataset locally under the `dataset` folder 
- Create a sample version of the dataset 
```bash

unzip 2019-Nov.csv.zip
cat 2019-Nov.csv | head -n 1000 > 202019-Nov-sample.csv
```

# Create an Amazon S3 bucket 

- Name of the Bucket : `ecommerce-raw-us-east-1-dev`
- Upload the dataset to Amazon S3 bucket 
```bash
aws s3 cp dataset/2019-Oct.csv.zip s3://ecommerce-raw-us-east-1-dev/ecomm_user_activity/p_year=2019/p_month=10/
aws s3 cp dataset/2019-Nov.csv.zip s3://ecommerce-raw-us-east-1-dev/ecomm_user_activity/p_year=2019/p_month=11/
aws s3 cp dataset/202019-Nov-sample.csv s3://ecommerce-raw-us-east-1-dev/ecomm_user_activity_sample/202019-Nov-sample.csv
aws s3 cp dataset/2019-Nov.csv s3://ecommerce-raw-us-east-1-dev/ecomm_user_activity_unconcompressed/p_year=2019/p_month=11/

```

#### ONLINE PROCESSING ######

# Create a Kinesis Data Stream 

- Data stream name : `ecommerce-raw-user-activity-stream-1`
- Capacity mode    : `On-demand`

# Start the eComm Workload simulation

- Edit the `ecomm-simulation-app/stream-data-app-simulation.py` with the bucket and key name 
- Execute the script 
```
python ecomm-simulation-app/stream-data-app-simulation.py
``` 

# Explore the Data Partition Key (EDA using AWS Glue) - How to find the right partition key for Kinesis 

- Create a Glue Crawler 
    - Name          : `ecomm-user-activity-crawler-1`
    - Data Source   : `s3://ecommerce-raw-us-east-1-dev/ecomm_user_activity_sample/202019-Nov-sample.csv`
    - Database Name : `my-db-ecomm`
- Run the Crawler `ecomm-user-activity-crawler-1`
- Open Athena to query the data 

```sql

SELECT category_id, count(*) FROM "my-db-ecomm"."ecomm_user_activity_sample"
GROUP BY category_id;

```

# Integration with Kinesis Data Analytics and Apache Flink 

- Creat a Amazon Kinesis Data Stream 
    - Data stream name : `ecommerce-raw-user-activity-stream-2`
    - Capacity mode    : `On-demand`
    
- Creat a Amazon Kinesis Data Analytics Streaming Application
    - Using Zeppelin Notebook 
    - Kinesis > Analytics Application > Studio > Create a Studio Notebook
    - Studio Notebook Name : `ecomm-streaming-app-v1` 
    - Choose the Source : `ecommerce-raw-user-activity-stream-1`
    - Choose the Destination : `ecommerce-raw-user-activity-stream-2`
    - Destination for code in Amazon S3 : `s3://ecommerce-raw-us-east-1-dev`

# Create the Apache Flink Application

- Modify the IAM role which automatically got create in the previous step and add the `AWSGlueServiceRole` policy 
    - IAM Role : `kinesis-analytics-ecomm-streaming-app-v1-us-east-1`
    - Permisions > Add Permission > Add `AWSGlueServiceRole`
- Run Studio notebook `ecomm-streaming-app-v1`
- Open the Zeppelin Notebook 
    - Upload the notebook : `sql-flink-ecomm-notebook-1`

- Create the table from the Zeppelin Notebook 

```sql

%flink.ssql

/*Option 'IF NOT EXISTS' can be used, to protect the existing Schema */
DROP TABLE IF EXISTS ecomm_user_activity_stream_1;

CREATE TABLE ecomm_user_activity_stream_1 (
  `event_time` VARCHAR(30), 
  `event_type` VARCHAR(30), 
  `product_id` BIGINT, 
  `category_id` BIGINT, 
  `category_code` VARCHAR(30), 
  `brand` VARCHAR(30), 
  `price` DOUBLE, 
  `user_id` BIGINT, 
  `user_session` VARCHAR(30),
  `txn_timestamp` TIMESTAMP(3),
  WATERMARK FOR txn_timestamp as txn_timestamp - INTERVAL '10' SECOND  
)
PARTITIONED BY (category_id)
WITH (
  'connector' = 'kinesis',
  'stream' = 'ecommerce-raw-user-activity-stream-1',
  'aws.region' = 'us-east-1',
  'scan.stream.initpos' = 'LATEST',
  'format' = 'json',
  'json.timestamp-format.standard' = 'ISO-8601'
);


/*Option 'IF NOT EXISTS' can be used, to protect the existing Schema */
DROP TABLE IF EXISTS ecomm_user_activity_stream_2;

CREATE TABLE ecomm_user_activity_stream_2 (
  `user_id` BIGINT, 
  `num_actions_per_watermark` BIGINT
)
WITH (
  'connector' = 'kinesis',
  'stream' = 'ecommerce-raw-user-activity-stream-2',
  'aws.region' = 'us-east-1',
  'format' = 'json',
  'json.timestamp-format.standard' = 'ISO-8601'
);

/* Inserting aggregation into Stream 2*/
insert into ecomm_user_activity_stream_2
select  user_id, count(1) as num_actions_per_watermark
from ecomm_user_activity_stream_1
group by tumble(txn_timestamp, INTERVAL '10' SECOND), user_id
having count(1) > 1;


```

- Push the data from Cloud9 instance using the `stream-data-app-simulation.py` script 
- Run the following query in Zeppelin

```sql

%flink.ssql(type=update)

select * from ecomm_user_activity_stream_1;

```

# Deply and run the Apache Flink Application

- Deploy the Flink Application from Zeppline Console 
  - Open the Zeppelin Notebook : `sql-flink-ecomm-notebook-1`
  - Click on `Deploy sql-flink-ecomm-notebook-1 as Kinesis Streaming Application`

- Once the application is deployed, edit the IAM role
  - Edit `kinesis-analytics-ecomm-streaming-app-v1-sql-flink-eco-us-east-1`
  - Permisions > Add Permission > Add the follow roles:
    - `AWSGlueServiceRole`
    - `AmazonKinesisFullAccess`
  - Run the application

- Open the Flink Dashboard 
- Go to Cloud 9 and push data 
- Monitor the data over Flink Dashboard 
  - You can also run the SQL query in the Zeppelin notebook 
    ```sql

    %flink.ssql(type=update)
    
    select * from ecomm_user_activity_stream_2;
    
    ```

# Create Lambda function to DynamoDB, CloudWatch and SNS (THIS IS of DDOS)

- Install the `aws_kinesis_agg` package 
    ```
    cd serverless-app 
    pip install aws_kinesis_agg -t .
    ```
- For more details check this [page](https://pypi.org/project/aws-kinesis-agg/)
  
- Build the lambda package and download the zip file. 
  
    ```
    zip -r ../lambda-package.zip .
    ```
- Upload the zip to S3

```
cd ..
aws s3 cp lambda-package.zip s3://ecommerce-raw-us-east-1-dev/src/lambda/
```
- Create the Lambda Function 
  - Name : `ecomm-detect-high-event-volume`
  - Python Version : 3.7 
  - Upload the code using the S3 location `s3://ecommerce-raw-us-east-1-dev/src/lambda/lambda-package.zip`
  - Set the permission > Lambda > Configuration > IAM Role and Add the following (This is not a best practice, but since this is for DEMO with sample dataset, its ok, as we are going to destroy all the resources after this workshop)
    - `CloudWatchFullAccess`
    - `AmazonDynamoDBFullAccess`
    - `AmazonKinesisFullAccess`
    - `AmazonSNSFullAccess`
  
  - Create the Environment Variable 
    - cloudwatch_metric	            : ecomm-user-high-volume-events  
    - cloudwatch_namespace	        : ecommerce-namespace-1         
    - dynamodb_control_table        : ddb-ecommerce-tab-1 
    - topic_arn                     : <PASTE HERE YOUR OWN TOPIC ARN>. [Create a SNS topic]
  
  - Add the Trigger  (This step would fail if you have not added the right permission `AmazonKinesisFullAccess`)
    - Kinesis : ecommerce-raw-user-activity-stream-2

# DynamoDB Data Modelling 

- Create a DynamoDB Table 
  - Name          :   `ddb-ecommerce-tab-1` [same as "dynamodb_control_table"]
  - Partition Key :   `ddb_partition_key`.  -> String
  - Secondary Key :   `ddb_sort_key`.       -> Number

# Lambda function walk-through 

- Add a test event of type `kinesis data stream` 
  
- Sample data [Decode/Encode from here](https://www.base64decode.org/)

```
Plain data  : {"user_id":101,"num_actions_per_watermark":10000} 
Encode date : eyJ1c2VyX2lkIjoxMDEsIm51bV9hY3Rpb25zX3Blcl93YXRlcm1hcmsiOjEwMDAwfQ== 

```

- Trigger the sample Event 

  - Verify the DynamoDB item 
  - Verify the lambda output 

# Deploy and Run Near Real Time End to End Data Pipeline 

- Make sure the Flink App is RUNNING 
- Push the Data from Cloud9 
- Wait for 5-10 secs
- Go to Lambda and checking the CloudWatch Logs 

#### BATCH PROCESSING ######

# Create a Glue Crawler 
  - With the data store as `s3://ecommerce-raw-us-east-1-dev/ecomm_user_activity_concompressed` 

# Use Athena to quert 

```sql

SELECT * FROM "my-db-ecomm"."ecomm_user_activity_unconcompressed" limit 10;

SELECT count(1)
FROM "my-db-ecomm"."ecomm_user_activity_unconcompressed" ;

```

# ETL using Glue Studio DataBrew and Apache Spark (This is for Optimization)


- ETL CSV to Parquet  (This job will take a long time)
- OPen AWS Glue > Glue Studio > Job > Name : `ecomm-ETL-Job`
- Select the follow:
  - Source S3 bucket : `s3://ecommerce-raw-us-east-1-dev/ecomm_user_activity_concompressed/`
  - Data Source Table : `ecomm_user_activity_unconcompressed`
  - Destination S3 bucket : `s3://ecommerce-raw-us-east-1-dev/ecomm_user_activity_parquet/`
  - Click on `Create a table in the Data Catalog and on subsequent runs, update the schema and add new partitions`
  - Destination Table : `ecomm_user_activity_parquet`
  - Select a Partition Key : `category_id`
  - Add the IAM Role under `Job details`
  - Run the JOB

# Use AWS Glue DataBrew to find the right partition key (Data Cardinality)

- Create a new DataSet 
  - Name : `ecomm_user_activity_unconcompressed-dataset-1`
  - Data Source : `s3://ecommerce-raw-us-east-1-dev/ecomm_user_activity_unconcompressed/`
- Create a Job
  - Job Output S3 location : `s3://ecommerce-raw-us-east-1-dev/`
  - IAM Role post-fix : `ecomm-iam-role-1`

# Scan the data and compare CSV and Parquet 

```sql

SELECT count(distinct category_id) from "ecomm_user_activity_unconcompressed";
SELECT count(distinct category_id) from "ecomm_user_activity_parquet";

```

# Explore the Glue DataBrew for Data Cardinality 

- Go to the `ecomm-user-activity-unconcompressed-dataset1` DataBrew Dataset 
- Explore the `Column Stats`

# Persisting All Raw Stream in S3 using Kinesis Firehose 

- Create a Kinesis Delivery Stream 
  - Source : Kinesis Data Stream 
  - Destination : Amazon S3 
  - S3 Location : `s3://ecommerce-raw-us-east-1-dev`
  - Buffer interval : `60`

- Start to push the data and check the S3 bucket after 3-4 mins 

# Build the dashboard using Quicksight

- Open [QuickSight dashboard](https://us-east-1.quicksight.aws.amazon.com/sn/start/analyses)
  - New Dataset : `sql_raw_user_activity` 
  
```sql
SELECT cast(event_time as timestamp) as event_time, event_type, product_id, category_code, brand, price, user_id, user_session, p_year, p_month, category_id
FROM "my-db-ecomm"."ecomm_user_activity_parquet";

SELECT count(distinct brand) from "my-db-ecomm"."ecomm_user_activity_parquet";
```

# Final Quicksight Dashboard using Spice 

- Create charts using the following dataset 
  - `sql_raw_user_activity`
  - `sql_raw_user_activity_filter_by_day` 
    [Create this dataset]

  ```sql
  SELECT  date_format(cast(event_time as timestamp), '%a' ) as Day,
        date_format(cast(event_time as timestamp), '%k' ) as Hour,
        event_type,
        category_code
  FROM "my-db-ecomm"."ecomm_user_activity_parquet" LIMIT 10;
  
  ```