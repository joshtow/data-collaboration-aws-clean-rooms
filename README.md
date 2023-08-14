# Data Collaboration with AWS Clean Rooms

This lab is provided as part of [AWS Innovate Data Edition](https://aws.amazon.com/events/aws-innovate/data/). Click [here](https://github.com/phonghuule/aws-innovate-data-edition-2023) to explore the full list of hands-on labs.

:information_source: You will run this lab in your own AWS account and running this lab will incur some costs. Please follow directions at the end of the lab to remove resources to avoid future costs.

## Overview

AWS Clean Rooms helps customers and their partners more easily and securely collaborate and analyze their collective datasets - without sharing or copying one another's underlying data. In this lab, we'll explore how to use AWS Clean Rooms to share data sets between two collaborating parties in different accounts to run join and aggregate queries without compromising the underlying PII and sensitive data.

## Architecture

In order to explore the data with Amazon Athena and Apache Spark, we'll create the following architecture.

![Architecture diagram](./images/architecture.png)

## Before You Begin
Before you start, there are a few things you should know:

- The resources created in this lab will have some costs associated with them. 
- You will need an IAM user or role with appropriate permissions to execute this lab. This includes permissions to create the resources defined in the CloudFormation template, as well as the manual activities performed through the AWS Management Console. 
- This lab does not rely on AWS Lake Formation for Glue catalog permissions; however, if Lake Formation is configured for the account, you will need to ensure the S3 bucket used for Clean Rooms is not managed by AWS Lake Formation.

## Pre-requisites

For this workshop, you will need two AWS accounts - one to represent the Insurance company and another for the Advertising company.

## Step 1 - Setup Insurance Account

1.1. Login to your Advertising account.

1.2. Download the CloudFormation template file here. This template will create the following in your account:
- S3Bucket - <accountid>-cleanroomslab-insurancedata - this bucket will be used to hold our Insurance data.
- IAM Role - <stackname>-GlueExecutionRole - this role will be used by AWS Glue to crawl the dataset.
- Glue Catalog Database - InsuranceDatabase - this database will hold the metadata for our data.
- Glue Crawler - <accountid>-cleanroomslab-crawler - this will create the table metadata in the AWS Glue data catalog.
- IAM Role - <stackname>-CleanRoomsServiceRole - this IAM role will be used by CleanRooms.

1.3. Log into your AWS Management Console, and navigate to AWS CloudFormation.

1.4. Click **Create stack**

1.5. Select **Upload a template file**, then choose the file you downloaded earlier and click **Next**.

1.6. Enter ```cleanroomslab``` as the **Stack name** and click **Next**.

1.7. On the **Configure stack options** page, leave the default values and click **Next**.

1.8. On the final page, scroll to the end and mark the checkbox next to **I acknowledge that AWS CloudFormation might create IAM resources with custom names.**, then click **Submit**.

1.9. Wait until the stack has completed successfully.

## Step 2 - Load Data and Run Pipeline

2.1. Navigate to **Cloud Shell** in the AWS Management Console. 
2.2. Execute the following statements.

```bash
accountid=$(aws sts get-caller-identity --query "Account" --output text)
aws s3 cp s3://TBD/data/customers/customers.csv ./customers.csv
aws s3 cp s3://TBD/data/customers/customers.csv ./policies.csv
aws s3 cp customers.csv s3://$accountid-cleanroomslab-insurancedata/data/customers/customers.csv
aws s3 cp policies.csv s3://$accountid-cleanroomslab-insurancedata/data/policies/policies.csv
```
2.3. Navigate to AWS glue in the AWS Management Console. 
2.4. Select **Crawlers** in the natigation pane, and select the **-cleanroomslab-crawler**.
2.5. Click **Run**, and wait for the crawler to complete - the state with be either **Ready** or **Stopping**.  Note that 2 tables should have been added to the Glue Data Catalog.

## Step 3 - Create Collaboration 
3.1. Navigate to [AWS Clean Rooms](https://us-east-1.console.aws.amazon.com/cleanrooms/home?region=us-east-1)
3.2. Click **Create collaboration**
3.3. Enter the following collaboration details and click **Next**

| Field  |      Values      |
|----------|:-------------:|
| Name |  :code[Insurance-Advertiser-Collaboration]{showCopyAction=true}| 
| Description |    :code[Insurance Advertiser Collaboration]{showCopyAction=true}   |  
| Member 1: Member display name| :code[Insurance Company]{showCopyAction=true} |
| Member 2: Member display name| :code[Advertising Company]{showCopyAction=true}|
| Member 2: Member AWS Account| AWS Account id of Advertising Company |
| Member Abilities: Member who can query and receive results| Account ID of the Advertising company|
| Query logging: Support query logging in this collaboration| check |

3.4. In **Configure membership**, turn on **Query Logging** and leave other configrations as default
3.5. Click **Next** and select **Create collaboration and membership**

## Step 4 - Create Configured Tables for the Insurance Data Set
Each configured table represents a reference to an existing table in the AWS Glue Data Catalog and data stored in Amazon S3 that has been configured for use in AWS Clean Rooms. A configured table contains an analysis rule that determines how the data can be used in AWS Clean Rooms queries.

4.1. Click on **Configured tables** in the Clean Rooms navigation pane.
4.2. Click **Configure new table**
4.3. Select **domain** database, table **customers**
4.4. In **Columns allowed in collaborations**, select **All columns** and click **Configure new table**
4.5. Repeat the steps above for **policies** table

## Step 5 - Configure the Customer analysis rule

5.1. Navigate to [Configured Tables](https://us-east-1.console.aws.amazon.com/cleanrooms/home?region=us-east-1#/configured-tables) and select **customers**
5.2. Select **Configure analysis rule**
5.3. Select **List** as **Type** and click **Next**
5.4. Choose **c_name** as the **Join Columns**
5.5. Under **List controls**, select the following columns:
- c_age
- c_city
- c_country
- c_gender
- c_postcode
- c_state
- c_customer_id
5.6. Click **Next**, then **Configure analysis rule**

## Step 6 - Configure the Polcies analysis rule

6.1. Navigate to [Configured Tables](https://us-east-1.console.aws.amazon.com/cleanrooms/home?region=us-east-1#/configured-tables) and select **policies**
6.2. Click **Configure analysis rule**
6.3. Leave **Aggregation** selected, and click **Next**
6.4. For the **Aggregate function**, choose **AVG**, and select the following columns:
- accidentyearbasicincurredamount
- accidentyearexcessincurredamount
- accidentyeartotalincurredamount
6.5. Under **Join controls**, select **Yes** under **Allow table to be queried by itself**.
6.6. Select **c_customer_id**.
6.7. Under **Dimension controls**, select **lineofbusiness**.
6.8. Select **Next**.
6.9. Under **Aggregation constraints**, select any of the aggregate columns (e.g. accidentyearbasicincurredamount) and enter a value of 100. 
6.10. Click **Next**, then **Configure analysis rule**.

## Step 7 - Associate configured tables to collaboration
7.1. On the **customers** table, click **Associate to collaboration**.
7.2. Select the **Insurance-Advertiser-Collaboration** and click **Choose collaboration**.
7.3. Under **Service access**, select **Create and use a new service role**.
7.4. Click **Associate table**.
7.5. Repeat for the **policies** table

### Step 8 - Setup Advertising Account

8.1. Log into the Advertising account.
8.2. Navigate to **Cloud Shell**, and execute the following statements

```bash
accountid=$(aws sts get-caller-identity --query "Account" --output text)
aws s3 cp s3://TBD/data/customers/customers.csv ./adimpressions.csv
aws s3 cp adimpressions.csv s3://#accountid-cleanroomslab-advdata/adimpressions/adimpressions.csv
```

8.3. Navigate to **AWS Glue** in the AWS Management Console.
8.4. Select **Crawlers** in the natigation pane, and select the **advertising-data-crawler**.
8.5. Click **Run**, and wait for the crawler to complete - the state with be either **Ready** or **Stopping**.  Note that 1 table should have been added to the Glue Data Catalog.

### Step 9 - Join the Collaboration

9.1. Navigate to **AWS Clean Rooms**, and select **Collaborations** from the navigation pane.
9.2. Click on the **Available to join** tab.
9.3. Select **Insurance-Advertiser-Collaboration** 
9.4. Review the collaboration details, and click **Create membership**
9.5. Make sure you turn on **Query logging** and **Create Membership**

## Step 10 - Associate AdImpressions table
10.1. Navigate to the **Configured tables**, and click **Configure new table**
10.2. Select **advertisingdomain** database, table **adimpressions**
10.3. In **Columns allowed in collaborations**, select **All columns** and click **Configure new table**

## Step 11 - Configure AdImpressions Analysis rule
11.1. Click **Configure analysis rule**
11.2. Select **List** as **Type** and click **Next**
11.3. Choose **c_name** as **Join Columns**
11.4. Click **Next** and **Configure analysis rule**

## Step 12 - Associate configured table to collaboration
12.1. On the **adimpresssions** table, click **Associate to collaboration**.
12.2. Select the **Insurance-Advertiser-Collaboration** and click **Choose collaboration**.
12.3. Under **Service access**, select **Create a new service role**.
12.4. Make sure you check the **Add a pre-configured policy with the necessary permissions to this role.** option.
12.5. Click **Associate table**.

## Step 13 - Query the Data
13.1 Select the **Insurance-Advertiser-Collaboration** collaboration.
13.2. Under **Tables**, you should be able to see the tables you're able to query.
13.3. Under the **Actions** dropdown, click **Set result settings**.
13.4. Browse and select the bucket ending with **-queries**.
13.5. Leave the format as **CSV** and click **Save changes**.
13.6. Execute the following query:

```
SELECT  AVG("policies"."accidentyearbasicincurredamount"),
	AVG("policies"."accidentyearexcessincurredamount"),
	AVG("policies"."accidentyeartotalincurredamount"),
	"policies"."lineofbusiness"

FROM  "policies"
GROUP BY "policies"."lineofbusiness"
```
This may take a several minutes as the query engine initiates after a cold start. Subsequent queries should be quicker.

13.7. Execute the following query:
```
SELECT DISTINCT "customers"."c_age",
		"customers"."c_city",
		"customers"."c_country",
		"customers"."c_gender",
		"customers"."c_postcode",
		"customers"."c_state"
FROM  "customers"
INNER JOIN  "adimpressions"
	ON  "customers"."c_name" = "adimpressions".c_name
```

## Conclusion

## Survey

Please complete the following survey to let us know how you found this lab and how we can improve.

## Clean Up

To clean up the resources created here, complete the following.
1. Delete all files from the **-insurancedata** and **-advdata** S3 buckets. 
2. Delete the AWS Clean Rooms collaboration.
3. Delete the CloudFormation templates in both accounts.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

