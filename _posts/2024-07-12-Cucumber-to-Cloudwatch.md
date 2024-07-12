---
layout: post
title: Leveraging AWS Lambda to Process Cucumber Test Results as CloudWatch Metrics
subtitle: "React, Express, Mongo, K8s, Prometheus, Grafana, K6"
date: 2024-07-12 18:30:13 -0000
description: "Use serverless computing with AWS Lambda to process your Serenity/Cucumber reports into usable metrics for quick alerting and dashboarding."
categories: [Guides]
tags: [DevOps, SRE, monitoring, cucumber, serenity, aws, lambda, cloudwatch]
pin: false
math: true
mermaid: true
image:
  path: /assets/img/posts/cucumbertocw/header.png
  lqip: /assets/img/posts/cucumbertocw/header_sqip.svg
  alt: 
---

# Leveraging AWS Lambda to Process Cucumber Test Results as CloudWatch Metrics

## Introduction

Site Reliability Engineers (SREs) often need a quick and straightforward way to determine if an environment is functioning as expected. While Serenity/Cucumber reports can be content-rich and provide extensive details, my approach of processing these test results as CloudWatch metrics offers a streamlined and efficient method for SREs to get immediate insights into the environment's health without sifting through detailed reports. This allows us to get alerted to the problem and respond quicker.

### What Are Serenity and Cucumber?

Serenity and Cucumber are powerful tools in the realm of automated testing. Cucumber is a testing framework that supports behavior-driven development (BDD). It allows you to write tests in a human-readable language (Gherkin), which makes it easier for non-technical stakeholders to understand the test scenarios. Serenity, on the other hand, extends Cucumber's capabilities by adding rich reporting and living documentation, making it easier to understand the state of your testing and requirements.

### AWS Lambda and CloudWatch Custom Metrics

AWS Lambda is a serverless computing service that runs your code in response to events and automatically manages the underlying compute resources for you. This makes it an ideal choice for executing tasks without provisioning or managing servers. AWS CloudWatch is a monitoring and observability service that provides data and actionable insights for AWS resources, applications, and services. Custom metrics in CloudWatch allow you to monitor specific metrics that are important to your applications, which can be crucial for gaining insights into application performance and operational health.

## Benefits of CloudWatch Metrics for Cucumber Test Results

Integrating Serenity/Cucumber test results with CloudWatch metrics brings several benefits:

1. **Automated Monitoring**: By having your test results as metrics in CloudWatch, you can set up automated alerts that notify you of test failures or performance regressions.
2. **Centralized Dashboard**: You can visualize your test results in CloudWatch dashboards, providing a centralized view of your application's health across different environments.
3. **Historical Data**: CloudWatch retains your metrics data, allowing you to analyze trends and patterns over time, which can be valuable for identifying recurring issues.

### Alternative Tools and Their Limitations

AWS Synthetic Canaries is an excellent tool for creating and managing synthetic tests that monitor your endpoints. However, this approach has limitations compared to using Cucumber tests. Synthetic Canaries typically simulate user interactions and basic endpoint checks, whereas Cucumber tests can cover complex user journeys and business logic. By processing Cucumber test results with AWS Lambda and visualizing them in CloudWatch, you get a more comprehensive and detailed view of your application's health.

## Guide to Processing Cucumber Test Results with AWS Lambda

Follow these steps to set up your environment and process Cucumber test results as CloudWatch metrics.

### Step 1: Create an S3 Bucket

1. Go to the S3 console.
2. Click "Create bucket".
![Create S3 Bucket](/assets/img/posts/cucumbertocw/s3-bucket-creation.png)
3. Choose any name for your bucket.
4. Block all public access.
5. Leave all other options as they are and click "Create bucket".
![Bucket Settings](/assets/img/posts/cucumbertocw/bucketconfig.png)


### Step 2: Create a Permissions Policy

1. Go to the IAM console.
2. Click "Policies" in the left navigation pane.
3. Click "Create policy".
4. Select the "JSON" tab.
5. Copy and paste the following JSON code into the editor:
    ```json
    {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "VisualEditor0",
        "Effect": "Allow",
        "Action": [
          "cloudwatch:TagResource",
          "cloudwatch:PutMetricData",
          "cloudwatch:UntagResource",
          "cloudwatch:ListMetrics"
        ],
        "Resource": "*"
      },
      {
        "Sid": "VisualEditor1",
        "Effect": "Allow",
        "Action": [
                  "s3:GetObject",
                  "s3:ListBucket"
              ],
        "Resource":  "arn:aws:s3:::your-bucket-name/*"
      }
    ]
  }
  ```

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

> Make sure to replace 'your-bucket-name' with the s3 bucket you have created.
{: .prompt-warning }

<!-- markdownlint-restore -->

![Create Permissions Policy](/assets/img/posts/cucumbertocw/policy_creation.png)

### Step 3: Create an Execution Role

1. Go to the IAM console.
2. Click "Roles" and then "Create role".
3. Choose "Lambda" as the trusted entity.
![Create Execution Role](/assets/img/posts/cucumbertocw/create_role.png)
4. Attach the policy created in Step 2.
5. Name the role and create it.


### Step 4: Create the Lambda Function

1. Go to the Lambda console.
2. Click "Create function".
3. Choose "Author from scratch".
4. Name your function and select Python as the runtime.
5. Choose x86_64 architecture.
6. Select "Use an existing role" and choose the role created in Step 3.
7. Deploy the function code provided.

``` python
import json
import boto3

def lambda_handler(event, context):
# Initialise the S3 and CloudWatch clients
    s3 = boto3.client('s3')
    cloudwatch = boto3.client('cloudwatch')

    # Get the bucket and key from the event
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']

    # Extract the environment from the file name
    file_name = key.split('/')[-1]
    environment = file_name.split('-')[0]

# Download the file from S3 to tmp
    s3.download_file(bucket,key,'/tmp/report.json')
# Open and read the report.json file

    with open('/tmp/report.json', 'r') as file:
        data = json.load(file)

    #Loop through the elements of the json file
    for entry in data:
        scenarios = entry['elements']
        all_steps_passed = True
        for scenario in scenarios:
            # Iterate over the steps in the scenario
            if 'steps' in scenario:
                steps = scenario['steps']
                step_number = 1
                for step in steps:
                    if 'result' in step:
                        print (step['name'])
                        result = step['result']
                        if 'status' in result:
                            status = result['status']
                            if status == 'passed':
                                cloudwatch.put_metric_data(
                                    Namespace=environment,
                                    MetricData=[
                                        {
                                            'MetricName': step['name'],
                                            'Dimensions': [
                                                {
                                                    'Name': 'Environment',
                                                    'Value': environment
                                                },
                                                {
                                                    'Name': 'Scenario',
                                                    'Value': scenario.get('name', 'Unnamed Scenario')
                                                },
                                                {
                                                    'Name': 'Step',
                                                    'Value': str(step_number)
                                                }
                                            ],
                                            'Value': 1,
                                        },
                                    ]
                                )
                            else:
                                all_steps_passed = False
                                cloudwatch.put_metric_data(
                                    Namespace=environment,
                                    MetricData=[
                                        {
                                            'MetricName': step['name'],
                                            'Dimensions': [
                                                {
                                                    'Name': 'Environment',
                                                    'Value': environment
                                                },
                                                {
                                                    'Name': 'Scenario',
                                                    'Value': scenario.get('name', 'Unnamed Scenario')
                                                },
                                                {
                                                    'Name': 'Step',
                                                    'Value': str(step_number)
                                                }
                                            ],
                                            'Value': 0,
                                        },
                                    ]
                                )
                        else:
                            print('No status found for step')
                    else:
                        print('No result found for step')
                    step_number += 1
            else:
                print('No steps found for scenario')
            scenario_name = scenario.get('name', 'Unnamed Scenario')
            if all_steps_passed:
                cloudwatch.put_metric_data(
                    Namespace=environment,
                    MetricData=[
                        {
                            'MetricName': scenario_name,
                            'Dimensions': [
                                {
                                    'Name': 'Environment',
                                    'Value': environment
                                },
                            ],
                            'Value': 1,
                        },
                    ]
                )
            else:
                cloudwatch.put_metric_data(
                    Namespace=environment,
                    MetricData=[
                        {
                            'MetricName': scenario_name,
                            'Dimensions': [
                                {
                                    'Name': 'Environment',
                                    'Value': environment
                                },
                            ],
                            'Value': 0,
                        },
                    ]
                )
```

![Create Lambda Function](/assets/img/posts/cucumbertocw/create_lambda.png)

### Step 5: Create the S3 Trigger

1. Go to your Lambda function.
2. Click "Add trigger".
3. Select "S3".
4. Choose the bucket created in Step 1.
5. Set the suffix to "report.json".

![Create S3 Trigger](/assets/img/posts/cucumbertocw/lambdaaddtrigger.png)

### Step 6: Naming the Cucumber File

Ensure your Cucumber JSON report file follows the naming convention `{environment}-report.json`. This will help segregate the metrics by environment in CloudWatch.

### Step 7: Upload JSON Result File

In your test pipeline, upload the JSON result file to the S3 bucket created earlier. This can be scheduled to run as frequently or infrequently as desired, allowing you to monitor the health of your environment and test results over time.

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

> You may likely have your own JSON report from cloudwatch, but if you are just following along, feel free to use the example one that I am using for this guide.
{: .prompt-info }

<!-- markdownlint-restore -->

```json
[
    {
      "uri": "features/ticket_selling.feature",
      "elements": [
        {
          "id": "ticket-selling-feature;user-navigates-to-homepage",
          "keyword": "Scenario",
          "name": "User navigates to homepage",
          "description": "",
          "line": 3,
          "type": "scenario",
          "steps": [
            {
              "keyword": "Given ",
              "name": "the user is on the login page",
              "line": 4,
              "match": {
                "location": "step_definitions/ticket_steps.rb:1"
              },
              "result": {
                "status": "passed",
                "duration": 1000000
              }
            },
            {
              "keyword": "When ",
              "name": "the user logs in",
              "line": 5,
              "match": {
                "location": "step_definitions/ticket_steps.rb:5"
              },
              "result": {
                "status": "passed",
                "duration": 2000000
              }
            },
            {
              "keyword": "Then ",
              "name": "the user should see the homepage",
              "line": 6,
              "match": {
                "location": "step_definitions/ticket_steps.rb:10"
              },
              "result": {
                "status": "passed",
                "duration": 3000000
              }
            }
          ]
        },
        {
          "id": "ticket-selling-feature;user-searches-for-events",
          "keyword": "Scenario",
          "name": "User searches for events",
          "description": "",
          "line": 10,
          "type": "scenario",
          "steps": [
            {
              "keyword": "Given ",
              "name": "the user is on the homepage",
              "line": 11,
              "match": {
                "location": "step_definitions/ticket_steps.rb:15"
              },
              "result": {
                "status": "passed",
                "duration": 1000000
              }
            },
            {
              "keyword": "When ",
              "name": "the user searches for 'concerts'",
              "line": 12,
              "match": {
                "location": "step_definitions/ticket_steps.rb:20"
              },
              "result": {
                "status": "passed",
                "duration": 2000000
              }
            },
            {
              "keyword": "Then ",
              "name": "the user should see a list of concerts",
              "line": 13,
              "match": {
                "location": "step_definitions/ticket_steps.rb:25"
              },
              "result": {
                "status": "passed",
                "duration": 3000000
              }
            }
          ]
        },
        {
          "id": "ticket-selling-feature;user-selects-an-event",
          "keyword": "Scenario",
          "name": "User selects an event",
          "description": "",
          "line": 17,
          "type": "scenario",
          "steps": [
            {
              "keyword": "Given ",
              "name": "the user is viewing a list of concerts",
              "line": 18,
              "match": {
                "location": "step_definitions/ticket_steps.rb:30"
              },
              "result": {
                "status": "passed",
                "duration": 1000000
              }
            },
            {
              "keyword": "When ",
              "name": "the user selects a concert",
              "line": 19,
              "match": {
                "location": "step_definitions/ticket_steps.rb:35"
              },
              "result": {
                "status": "passed",
                "duration": 2000000
              }
            },
            {
              "keyword": "Then ",
              "name": "the user should see the concert details",
              "line": 20,
              "match": {
                "location": "step_definitions/ticket_steps.rb:40"
              },
              "result": {
                "status": "failed",
                "error_message": "Concert details not displayed",
                "duration": 3000000
              }
            }
          ]
        }
      ],
      "name": "Ticket Selling Feature",
      "description": "This feature describes the navigation around a ticket selling website",
      "id": "ticket-selling-feature",
      "keyword": "Feature",
      "line": 1
    }
  ]
  ```

### Step 8: View CloudWatch Metrics, Create Dashboards & Alerts

1. Go to the CloudWatch console.
2. Click on "Metrics" in the left navigation pane.
3. Select the namespace where your metrics are stored (e.g., `/aws/lambda/your-lambda-function-name`).
4. You should see the metrics generated by your Lambda function.
5. Click on the metric name to view detailed graphs and statistics.
6. Use the available options to set up alarms or dashboards as needed to monitor your test results.

![Check CloudWatch Metrics](/assets/img/posts/cucumbertocw/cloudwatch_metrics.png)


<!-- markdownlint-capture -->
<!-- markdownlint-disable -->

> Now your test data is in CloudWatch, its up to you what you do with it! Create dashboards or alerts to help notify your team to environment issues, and allow them to respond quickly.
{: .prompt-tip }

<!-- markdownlint-restore -->

## Conclusion

Integrating Serenity/Cucumber test results with CloudWatch via AWS Lambda provides a robust solution for automated monitoring and alerting. This approach not only offers a detailed and comprehensive view of your application’s health but also leverages the power of AWS’s serverless and monitoring capabilities. Compared to alternatives like AWS Synthetic Canaries, this method gives you the flexibility and depth needed for complex testing scenarios. 

