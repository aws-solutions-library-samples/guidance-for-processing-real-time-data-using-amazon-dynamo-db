# Guidance for processing real time data using Amazon Dynamodb

## Table of Content (required)

List the top-level sections of the README template, along with a hyperlink to the specific section.

### Required

1. [Overview](#overview)
    - [Cost](#cost)
2. [Prerequisites](#prerequisites)
    - [Operating Systems](#operating-systems)
3. [Deployment Steps](#deployment-steps)
4. [Deployment Validation](#deployment-validation)
5. [Running the Guidance](#running-the-guidance)
6. [Next Steps](#next-steps)
7. [Cleanup](#cleanup)

## Overview

This AWS Guidance is built to showcase how to perform aggregations on Amazon DynamoDB tables using Amazon DynamoDB Streams and AWS Lambda. It addresses the challenge of efficiently aggregating data in DynamoDB tables, which is crucial for applications needing to perform operations like counting, summing, or determining max/min values directly on DynamoDB without exporting data to other services or performing costly table scans. This solution leverages the real-time processing capability of DynamoDB Streams coupled with the compute power of AWS Lambda to efficiently aggregate data, offering a method to handle real-time data aggregation needs within DynamoDB itself.

For more detailed information, please visit the AWS Database Blog post​ http://disq.us/t/4j3itk0.

Architecture diagram:

![Architecture diagram](./assets/images/architecture.png)

### Cost 
You are responsible for the cost of the AWS services used while running this Guidance.

As of 04/15/2024, the cost for running this guidance with the default settings in the US East (N. Virginia) is approximately $48.27 per month for processing 1 million items.


| AWS service  | Dimensions | Monthly Cost [USD] |
| ----------- | ------------ | ------------ |
| Amazon DynamoDB | Table class (Standard), Average item size (1 KB), Data storage size (100 GB), Number of Writes and reads (Eventually consistent) per month (1 million)  | $ 26.38  |
| AWS Lambda | 1 Million requests per month, Execution time of 20 milliseconds, 128 MB of memory allocated, Concurrency of 100  | $ 30.46  |
| Total |  | $ 56.84 |

We recommend creating a [Budget](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html) through [AWS Cost Explorer](https://aws.amazon.com/aws-cost-management/aws-cost-explorer/) to help manage costs. Prices are subject to change. For full details, refer to the pricing webpage for each AWS service used in this Guidance.

## Prerequisites

- The [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed.
- The visual editor of your choice, for example [Visual Studio Code](https://code.visualstudio.com/).
- Install CloudFormation template to quickly deploy and test the solution. Follow the steps that are mentioned in this [link](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-create-stack.html) to deploy the stack.
- Permissions to deploy an DynamoDB table and an AWS Lambda function.

## Operating systems

This solution supports build environments in Mac or Windows.

### AWS account requirements 
This deployment requires that you have access to the following AWS services:

- Amazon API Gateway
- Amazon DynamoDB
- AWS Lambda

## Deployment Steps

These deployment instructions are optimized to best work on Mac or Amazon Linux 2023. Deployment in another OS may require additional steps.

1. Clone the repo using command ``` git clone https://github.com/aws-solutions-library-samples/guidance-for-building-aggregations-for-dynamodb-tables-using-amazon-dynamodb-streams-on-aws.git```
2. cd to the repo folder ```cd guidance-for-building-aggregations-for-dynamodb-tables-using-amazon-dynamodb-streams-on-aws```
3. cd to deployment folder to deploy the CloudFormation template ```cd deployment```
4. Run the below command to deploy the stack in your account.
```
aws cloudformation create-stack \
--stack-name ddbaggregate \
--template-body file://CloudFormation.yaml \
--capabilities CAPABILITY_NAMED_IAM
```

## Deployment Validation 

Open CloudFormation console and verify the status of the template with the name of the stack specified in step 4 of the deployment steps.

## Running the Guidance

Once the CloudFormation stack is deployed, Follow the below steps to test the guidence.

1. Insert test items to the source DynamoDB table `(Order_by_item)`.
   
For Linux or Mac:
```
     aws dynamodb put-item --table-name Order_by_item \
     --item '{"orderid": {"S": "178526"}, "order_date": {"S": "'"$(date -u +"%Y-%m-%dT%H:%M:%SZ")"'"}, "item_number": {"S": "item123"}, "quantity": {"N": "10"}}'
     
     aws dynamodb put-item --table-name Order_by_item \
     --item '{"orderid": {"S": "178527"}, "order_date": {"S": "'"$(date -u +"%Y-%m-%dT%H:%M:%SZ")"'"}, "item_number": {"S": "item123"}, "quantity": {"N": "30"}}'

     aws dynamodb put-item --table-name Order_by_item \
     --item '{"orderid": {"S": "172528"}, "order_date": {"S": "'"$(date -u +"%Y-%m-%dT%H:%M:%SZ")"'"}, "item_number": {"S": "item312"}, "quantity": {"N": "10"}}'
     
     aws dynamodb put-item --table-name Order_by_item \
     --item '{"orderid": {"S": "178529"}, "order_date": {"S": "'"$(date -u +"%Y-%m-%dT%H:%M:%SZ")"'"}, "item_number": {"S": "item312"}, "quantity": {"N": "30"}}'
```

For Windows:

```
    aws dynamodb put-item --table-name Order_by_item ^
     --item '{"orderid": {"S": "178526"}, "order_date": {"S": "'"$(date -u +"%Y-%m-%dT%H:%M:%SZ")"'"}, "item_number": {"S": "item123"}, "quantity": {"N": "10"}}'
     
     aws dynamodb put-item --table-name Order_by_item ^
     --item '{"orderid": {"S": "178527"}, "order_date": {"S": "'"$(date -u +"%Y-%m-%dT%H:%M:%SZ")"'"}, "item_number": {"S": "item123"}, "quantity": {"N": "30"}}'

     aws dynamodb put-item --table-name Order_by_item ^
     --item '{"orderid": {"S": "178528"}, "order_date": {"S": "'"$(date -u +"%Y-%m-%dT%H:%M:%SZ")"'"}, "item_number": {"S": "item312"}, "quantity": {"N": "10"}}'
     
     aws dynamodb put-item --table-name Order_by_item ^
     --item '{"orderid": {"S": "178529"}, "order_date": {"S": "'"$(date -u +"%Y-%m-%dT%H:%M:%SZ")"'"}, "item_number": {"S": "item312"}, "quantity": {"N": "30"}}'
```
The following image shows how the source DynamoDB table (`Order_by_item`) data would look like after the insert statements in NoSQL Workbench.

![testpic1](./assets/images/TestPic1.png)

2. Verify if the target DynamoDB table `(item_count_by_date)`  has the aggregated data using AWS CLI:
 For Linux or Mac:
 ```
aws dynamodb query --table-name item_count_by_date \
    --key-condition-expression "item_number = :pkValue" \
    --expression-attribute-values '{":pkValue": {"S": "item123"}}' \
    --projection-expression "item_number, quantity"
```
For Windows:
```
 aws dynamodb query --table-name item_count_by_date ^
    --key-condition-expression "item_number = :pkValue" ^
    --expression-attribute-values '{":pkValue": {"S": "item123"}}' ^
    --projection-expression "item_number, quantity"
```

Response:
```
{
    "Items": [
        {
            "quantity": {
                "N": "40"
            },
            "item_number": {
                "S": "item123"
            }
        }
    ],
    "Count": 1,
    "ScannedCount": 1,
    "ConsumedCapacity": null
}
```

This image shows the target DynamoDB table `(item_count_by_date)`, which contains the aggregated data, as viewed in NoSQL Workbench.

![testpic2](./assets/images/TestPic2.png)

## Next Steps

Having explored how to execute one of the aggregate functions, addition, you can now adjust the Lambda function to experiment with and carry out other aggregate functions. If your use case aligns with the example provided in the guidance, simply update the table names in the Lambda function to begin performing the aggregations.

## Cleanup

1. To delete the stack deployed using the CloudFormation template follow the steps mentioned in this [link](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-delete-stack.html).
2. If using AWS Cli run the following command: 
   ```
   aws cloudformation delete-stack --stack-name ddbaggregate
   ```


## Notices

*Customers are responsible for making their own independent assessment of the information in this Guidance. This Guidance: (a) is for informational purposes only, (b) represents AWS current product offerings and practices, which are subject to change without notice, and (c) does not create any commitments or assurances from AWS and its affiliates, suppliers or licensors. AWS products or services are provided “as is” without warranties, representations, or conditions of any kind, whether express or implied. AWS responsibilities and liabilities to its customers are controlled by AWS agreements, and this Guidance is not part of, nor does it modify, any agreement between AWS and its customers.*
