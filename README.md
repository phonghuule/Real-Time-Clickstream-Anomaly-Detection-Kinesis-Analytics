# Real-Time Clickstream Anomaly Detection Kinesis Analytics

This lab is provided as part of **[AWS Innovate Data Edition](https://aws.amazon.com/events/aws-innovate/data/)**,  it has been adapted from an [AWS Workshop](https://aws-dataengineering-day.workshop.aws/300/320-main-lab.html)
Click [here](https://github.com/phonghuule/aws-innovate-data) to explore the full list of hands-on labs.

ℹ️ You will run this lab in your own AWS account and running this lab will incur some costs. Please follow directions at the end of the lab to remove resources to avoid future costs.

## Table of Contents  
* [Overview](#overview)  
* [Architecture](#architecture)  
* [Pre-Requisites](#pre-Requisites)  
* [Setting up Amazon Kinesis Data Generator Tool](#setting-up-amazon-kinesis-data-generator-tool)  
* [Kinesis Analytics Pipeline Application setup](#kinesis-analytics-pipeline-application-setup)
* [Configure Output stream to a Destination](#configure-output-stream-to-a-destination)
* [Test E2E Architecture](#test-e2e-architecture)
* [Cleanup](#cleanup)
* [Survey](#survey)


## Overview
This lab helps you to use Kinesis Data Analytics to perform Real-Time Clickstream Anomaly Detection. The use case demonstrated in this lab is analysing clickstream data. Amazon Kinesis Data Analytics is a managed service that makes it easy to analyse web log traffic that drive business decisions in real-time.
In this lab there are a few digital ad concepts in use:
#### Impression
An impression (also known as a view-through) is when a user sees an advertisement. In practice, an impression occurs any time a user opens an app or website and an advertisement is visible.
#### Click/Engagement
A Click or an Engagement is when a customer clicks on an ad.
#### Click through Rate (CTR)
A CTR is computed as Clicks / Impressions * 100. A CTR is a measure of the ad's effectiveness. While digital marketers are interested in the CTR to determine the performance of the ad, they are also interested if anamolous low-end CTRs that could be a result of a bad image or bidding model.

This lab helps identify anomalous data points within a data set using an unsupervised learning algorithm [RANDOM_CUT_FOREST](https://docs.aws.amazon.com/sagemaker/latest/dg/randomcutforest.html). 

The lab uses the Kinesis Data Generator (KDG) tool to generate data and sends it to Amazon Kinesis. The tool provides a user-friendly UI that runs directly in your browser. 

**Duration** - Approximately 2 hours

## Architecture

![Architecture](./images/architecture.png)

## Pre-Requisites

The below cloudformation template deploys the lab pre-requisites in your AWS account. 
Note:
* The stack selects the Asia Pacific(Sydney) for deployment, please alter the region as required.
* Deploy stack

    | Region | CloudFormation |
    | --- | --- |
    | Asia Pacific(Sydney) | [![Launch Stack](images/cloudformation-launch-stack-button.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-southeast-2#/stacks/new?stackName=rtcs-anamoly-detect&templateURL=https://s3.amazonaws.com/aws-dataengineering-day.workshop.aws/Kinesis_PreLab.yaml)|

* The CloudFormation stack takes the following parameters as input:
    |Parameter|Description|
    | --- | --- |
    |Username|User name for the KDG tool. Enter ```kplsetup```. This will be a user in the Amazon Cognito|
    |Password|Password for the user in Amazon Cognito|
    |Email|The email address to send anomaly events|
    |SMS|The mobile phone number to send anomaly events|

* Acknowledge that the CloudFormation might create IAM resources
* While the stack runs, watch out for an email as below.
![EmailSubscription](./images/EmailSubscription.png)
* Confirm the subscription to receive the below message.
![EmailSubscriptionConfirmation](./images/EmailSubscriptionConfirmation.png)
* Once the stack is deployed, click the outputs tab to view more information.
    * **KinesisDataGeneratorUrl** - This value is the Kinesis Data Generator (KDG) URL.
    * **RawBucketName** - Store raw data coming from KDG.
    * **ProcessedBucketName** - Store transformed data 
   ![Stackoutput](./images/Stackoutput.png) 

## Setting up Amazon Kinesis Data Generator Tool
On the Outputs tab of the deployed CloudFormation stack, note the Kinesis Data Generator URL. Navigate to this URL and login into the Amazon Kinesis Data Generator (KDG) Tool.
The KDG simplifies the task of generating data and sending it to Amazon Kinesis. The tool provides a user-friendly UI that runs directly in your browser. With the KDG, you can do the following tasks:
   * Create templates that represent records for specific use cases
   * Populate the templates with fixed or random data
   * Save the templates for future use
   * Continuously send thousands of records per second to your Amazon Kinesis stream or Firehose delivery stream

#### Kinesis Data Generator (KDG) and Template setup
1. Click on the KinesisDataGeneratorUrl on the Outputs tab on the stack.
2. Sign in using the username and password you enetered in the CloudFormation console.
![KDGLogin](./images/KDGLogin.png)
3. After you sign in, you should see the KDG console. Select the region where the stack was deployed. e.g. ap-southeast-2.
4. You need to set up some templates to mimic the clickstream web payload.
5. Create the following templates in the KDG console. But do not click on the 'Send Data' yet:
   |**Records per Second**|**Template Name**|**Payload**|**Description**|
   | --- | --- | --- | --- |
   | 1 |```Schema Discovery Payload``` | ```  {"browseraction":"DiscoveryKinesisTest", "site": "yourwebsiteurl.domain.com"}``` |Payload used for schema discovery|
   | 1 |```Click Payload``` | ```  {"browseraction":"Click", "site": "yourwebsiteurl.domain.com"} ``` |Payload used for clickstream data|
   | 1 |```Impression Payload``` | ```  {"browseraction":"Impression", "site": "yourwebsiteurl.domain.com"}``` |Payload used for impression stream data|
   
![KDGTemplateSetup](./images/KDGTemplateSetup.png)

 
## Kinesis Analytics Pipeline Application setup
1. Navigate to the [Amazon Kinesis](https://ap-southeast-2.console.aws.amazon.com/kinesis/home?region=ap-southeast-2#/home) console
2. On the Get Started portlet, select Kinesis Data Analytics, and click on Create Application.
     ![KinesisDataAnalyticsGetStarted](./images/KinesisDataAnalyticsGetStarted.png)
3. On the Kinesis Data Analytics - Create application page:
    * Select Legacy SQL as the runtime and click on Create Application.
            ![selectRuntime](./images/selectRuntime.png)
    * For application name, type ```anomaly-detection-application```.
    * Type a description for the analytics application and click on Create Application
    * Once created, Open analytics application and expand the 'Steps to configure application' and click on 'Configure source stream'.
            ![ConfigureSourceStream](./images/ConfigureSourceStream.png)
    * Select the Source as Kinesis Data Firehose delivery stream and select the Delivery Stream created by the CloudFormation template: **{stack-name}-FirehoseDeliveryStream-{random string}**
    * Select 'Off' for the Record preprocessing with AWS Lambda for this lab.
    * Select 'Choose from IAM roles that Kinesis Data Analytics can assume' and associate the following service role: **{stack-name}-CSEKinesisAnalyticsRole-{random string}**
    * **Do not** Click on Discover Schema at this point.
        Open the KDG tool, and click on 'Send Data' for the 'Schema Discovery Payload' template. This will send the sample data to the Kinesis Data Firehose delivery stream.
        Make sure to login to the tool. 
            ![senddatatokinesis](./images/senddatatokinesis.png)
    * Now go back to the AWS console and click on Discover Schema to display the schema and the formatted data received. 
            ![disoverschema](./images/disoverschema.png)
    * Leave the 'Use discovered schema' option selected for 'Schema options' and click on Save changes. The source stream should show as below:
            ![source-da](./images/source-da.png)
    * Now navigate to the 'Real-time analytics' tab and click on Configure.
    * Replace the sql with the contents of the following file: [anomaly_detection.sql](https://aws-dataengineering-day.workshop.aws/300/scripts/Kinesis_Anlaytics_anomaly_detection.sql) and click on 'Save and run application'. This should take a minute to run.
            ![rtanalyticssql](./images/rtanalyticssql.png)
    
    * On the same page scroll down to view the Input and Output sub tab. The Input sub tab will list the source stream and the Output sub-tab will display the output streams created by the SQL run before.
            ![inputoutput](./images/inputoutput.png)

At this point you have created:
* Kinesis Data Analytics application schema for click stream and impression stream data using the 'Schema Discovery Payload' sent from the KDG tool
* An output stream schema, using [anomaly_detection.sql](https://aws-dataengineering-day.workshop.aws/300/scripts/Kinesis_Anlaytics_anomaly_detection.sql). The schema will in real-time convert the input stream to the output stream.

Note: The DESTINATION_SQL_STREAM uses the RANDOM_CUT_FOREST function to detect anomalies within the input data stream. You can read more about the function [here](https://docs.aws.amazon.com/kinesisanalytics/latest/sqlref/sqlrf-random-cut-forest.html).

## Configure Output stream to a Destination
The DESTINATION_SQL_STREAM output stream contains records where anomalies have been found and need to be reported. In this section we will configure a new destination (a lambda function) to direct the anomalies to an SNS Topic.
1. Stop sending data configured in the previous [step](#setting-up-amazon-kinesis-data-generator-tool).
2. In the output subtab click on 'Connect to destination'.
            ![outputconnecttodest](./images/outputconnecttodest.png)
2. Select 'AWS Lambda function' as a destination and browse for the Lamdba function,**CSEBeconAnomalyResponse**, created by the CloudFormation template with version set as $LATEST.
3. For Access Permissions, select 'Choose from IAM roles that Kinesis Data Analytics can assume' and select  **{stack-name}-CSEKinesisAnalyticsRole-{random-string}** as the service role.
            ![destinationlambda](./images/destinationlambda.png)
4. For the In-application stream name, select 'Choose an existing in-application stream' and select 'DESTINATION_SQL_STREAM' and leave the Ouput format as JSON. Click on Save Changes.
            ![inappstreamname](./images/inappstreamname.png)
            ![lamdadest](./images/lamdadest.png)

## Test E2E Architecture
This section tests the end to end architecture. The KDG tool is configured to send Impression and multiple click stream data to imitate an anomaly. The Kinesis Data Analytics application detects anomalies and sends these anomalies to the Lamdba function. The Lamdba function then sends a notification to an SNS topic and in turn to an email and an sms.
**Note:** Stop data the click stream data after 20-30 seconds to prevent a large number of sms being sent.

1. To send in the click and impression data we will need to login to the KDG tool in **5 tabs** and Send data to the Kinesis data stream firehose based on the table below. **Note:** Follow notes for Tab 3,4,5 to prevent excess notifications:
    |**Tab**|**Template**|**Records per Second**|**Payload**|**Notes**
    | --- | --- | --- | --- | --- |
    |1|Impression|1| ```  {"browseraction":"Impression", "site": "yourwebsiteurl.domain.com"}``` |Can leave running|
    |2|Click|1| ```  {"browseraction":"Click", "site": "yourwebsiteurl.domain.com"} ``` |Can leave running|
    |3|Click|1| ```  {"browseraction":"Click", "site": "yourwebsiteurl.domain.com"} ``` |Stop after 20-30 seconds to prevent large number of emails and sms from SNS Topic|
    |4|Click|1| ```  {"browseraction":"Click", "site": "yourwebsiteurl.domain.com"} ``` |Stop after 20-30 seconds to prevent large number of emails and sms from SNS Topic|
    |5|Click|1| ```  {"browseraction":"Click", "site": "yourwebsiteurl.domain.com"} ``` |Stop after 20-30 seconds to prevent large number of emails and sms from SNS Topic|
    
2. With the KDG tool sending data to Kinesis, the input data can be viewed in the AWS Console.
            ![inputclickimpressionstream](./images/inputclickimpressionstream.png)
4. Similarly the data can be viewed in the output data stream. E.g. IMPRESSIONSTREAM, CLICKSTREAM. DESTINATION_SQL_STREAM. 
            ![impressionstream](./images/impressionstream.png) ![outputclickstream](./images/outputclickstream.png)![destinationanomalyscore](./images/destinationanomalyscore.png)
5. An email and sms is sent based on the detected anomalies. These anomalies are based on the 'DESTINATION_SQL_STREAM' stream with a destination of the lamda function.
            ![anomalyemail](./images/anomalyemail.png)

## Cleanup
Follow the below steps to cleanup your account to prevent any aditional charges:
* Navigate to the Kinesis Data Analytics application.
* Select the 'anomaly-detection-application' application and click on Stop.
            ![stopanalyticsapplication](./images/stopanalyticsapplication.png)
* Select the 'anomaly-detection-application' application and click on Delete.
            ![deleteanalyticsapplication](./images/deleteanalyticsapplication.png)
* Navigate to the CloudFormation and find the stack that was deployed as part of the pre-requisites.
* Select the stack and delete. This action will delete all the resources that were created as part of the lab and stop incurring charges.
            ![deletedeployedstack](./images/deletedeployedstack.png)
            
## Survey
Please help us to provide your feedback [here](https://amazonmr.au1.qualtrics.com/jfe/form/SV_3a6rNirgLrWYRW6?Session=HOL08). Participants who complete the surveys from AWS Innovate Online Conference - Data Edition will receive a gift code for USD25 in AWS credits. AWS credits will be sent via email by 30 September, 2021.
