# Batch-ETL-Using-AWS-EMR-in-Managed-Airflow

## Business Problem

A company has given the data team its data on job postings. The data scientist's task is to predict the salary of the job postings and has determined to use the average salary as the baseline model for each categorial features. Having to manually unpdate this process each time new data is introduced requires immediate attention before he can move on to other analytic tasks. My job as the data engineer is to create a pipeline that prepares the summary for the average salary of each categorical features; then the data scientist can take the output result and proceeds to perform data analysis and training Machine Learning model.

## Introduction

**Datasets**: The source data consists of three CSV files and are stored on Cloud in an Amazon S3 bucket. Each observation represents an individual's job posting; each column represents unique information about this applicant and the job applied to.
1. [train_features.csv](data/train_features.csv)
2. [test_features.csv](data/test_features.csv)
3. [train_salaries.csv](data/train_salaries.csv)

![](images/data.PNG)

**Amazon Managed Apache Airflow (MWAA)**: A service hosted on AWS that manage Apache Airflow on the server side. This takes away the user's responsibility in repetitively configuring the airflow environment, which can be unnecessarily time-consuming and mundane. I decided to launch an airflow environment on Amazon MWAA so I can manage the data pipeline without having to worry about the underlying hardware configuration. 

**Amazon Elastic MapReduce (EMR)**: Amazon EMR can be used to process a large amount of data using tools such as Apache Hadoop/Spark. The user can easily provision resources for the spark clusters in a highly scalable Big Data environment. 

**Amazon CloudFormation**: A service that provisions a set of AWS infrastructure resources in a reuseable way through a template. For example, you can launch multiple AWS services simultaneously by specifying the corresponding configuration.

## Goal
Launch an Amazon MWAA environment to create a data pipeline that orchestrates a batch ETL processing workflow in Amazon EMR.

## Architecture Overview
At a high level, the AWS cloud environment for this project is illustrated below. 

![](images/architecture_overview.png)
 
An Amazon MWAA environment requires the following resources:
- a VPC that spans across 2 different availability zones, each of which consists of a public and a private subnet, respectively. 
- a NAT gateway with a route table in each public subnet to connect to the internet 
- A S3 gateway VPC endpoint in each availability zone to ensure private connection between Amazon MWAA and Amazon S3.
- An EMR interface VPC endpoint in each availability zone to ensure secure connection to Amazon EMR from Amazon MWAA.

All the above resources are provisioned with the template [airflow_cft.yml](airflow_cft.yml) using Amazon CloudFormation. (The template is retrieved from the AWS Big Data Blog [Orchestrating analytics jobs on Amazon EMR Notebooks using Amazon MWAA](https://aws.amazon.com/blogs/big-data/orchestrating-analytics-jobs-on-amazon-emr-notebooks-using-amazon-mwaa/). You may also create the same template using Amazon CloudFormation template designer if you want to gain some practice) These are essential in properly setting up the airflow environment in AWS MWAA. Reference the workflow diagram below for a clearer illustration. 

## Workflow Diagram

![](images/pipeline_design.png)

## Pipeline Design

![](images/salary_pipeline_dag_graph.PNG)

At a high-level, the data pipeline orchestrates the following tasks:
1. Trigger the DAG
2. Provision an EMR cluster
3. Submit a spark step in the EMR cluster nodes that executes the ETL workflow 
4. Wait for the spark submission to complete
5. Terminate the EMR cluster
6. End the DAG

## Getting Started

***1. Create an EC2 instance keypair***

- The spark cluster created by Amazon EMR launches EC2 instances; you will need a keypair for secure access into the instances.
- Go to Amazon EC2 and scroll down to the Network & Security section
- Create a keypair in .pem format called NV-keypair (or another name you like; but make sure you manually change the keypair name in [DAG.py](dags/DAG.py).)
- The EMR_EC2_DefaultRole and EMR_DefaultRole will be automatically created for you by AWS. These IAM roles allow your EC2 instances in the spark cluster to assume the role necessary to access and work with Amazon EMR

![](images/dag_spark_config.PNG)

***2. Create an Amazon S3 bucket***

- Amazon MWAA isn't available across all regions. Make sure you select a region where the service is actually available.
- Make sure the bucket name starts with 'airflow-' in order to be compatible with the template we will use later on. (You may need to change the bucket name in [DAG.py](dags/DAG.py) and [avg_sal_etl.py](avg_sal_etl.py).)
- Create a folder named data and upload the files [train_features.csv](data/train_features.csv), [train_salaries.csv](data/train_salaries.csv), and [test_features.csv](data/test_features.csv). 
- Create a folder named dags and upload the [DAG.py](dags/DAG.py) file. You must put the file in the dags folder since the airflow environment will look specifically for this folder.
- Create a folder named etl and upload the [avg_sal_etl.py](avg_sal_etl.py) file.
- Upload the [airflow_cft.yml](airflow_cft.yml) template. Amazon CloudFormation will look for this template in the bucket.

![](images/S3_bucket_prerequisites.PNG)

***3. Launch the MWAA airflow environment in CloudFormation***

- Copy and paste the object URL of [airflow_cft.yml](airflow_cft.yml) under Amazon S3 URL block.
- Finish naming the CloudFormation stack and wait for the template to complete; it should take around 20 minutes.

![](images/cloudformation_template.PNG)

***4. Go to Amazon Managed Apache Airflow and open the airflow UI***

- Turn on to start the DAG. Manually trigger the DAG if necessary. 
- The airflow DAG runs on the DAG.py file in s3://airflow-salary-prediction-de/dags/

![](images/airflow_dag.PNG)

***5. Go to Amazon EMR***

- Find the 'salary-prediction-emr' spark cluster that is running. 
- Go to the step panel and double check that the average_salary step is in queue waiting to be executed.

![](images/spark_step.PNG)

- The DAG schedule interval for the DAG is `0 0 0 * *`; if the DAG keeps staying ON it will be triggered once every day at 12:00 AM. Once the Spark step is completed, we should see that all the steps succeeded like in the follwing tree view in the airflow UI

![](images/salary_pipeline_dag_tree.PNG)

***6. Check the output***

- Go to the S3 bucket and check the output folder.

![](images/output.PNG)

- The Spark job executed in Amazon EMR calculates the average salary of each group within each categorical variables. For example, the following output is the average salary of each industry

![](images/output_example.PNG)

***7. Clean up the environment***

- Amazon EMR and MWAA do incur some charges so it is essential to end these services when not in use.
- Go to Amazon CloudFormation and delete the template. This step will delete all the resources you created (e.g VPC, endpoints, NAT gateway, airflow environment, etc).
- Go to Amazon S3 and delete the bucket. 

# References

1. <https://aws.amazon.com/blogs/big-data/orchestrating-analytics-jobs-on-amazon-emr-notebooks-using-amazon-mwaa/>
2. <https://docs.aws.amazon.com/mwaa/latest/userguide/samples-emr.html>
3. <https://aws.amazon.com/emr/?whats-new-cards.sort-by=item.additionalFields.postDateTime&whats-new-cards.sort-order=desc>
4. <https://www.startdataengineering.com/post/how-to-submit-spark-jobs-to-emr-cluster-from-airflow/>
5. <https://airflow.apache.org/docs/apache-airflow-providers-amazon/stable/_modules/airflow/providers/amazon/aws/example_dags/example_emr_job_flow_manual_steps.html>
6. <https://airflow.apache.org/docs/apache-airflow/1.10.12/_api/airflow/contrib/operators/index.html>









