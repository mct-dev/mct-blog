---
title: Netflix Voting Service - Case Study
description: Building a voting service to solve a fictitious enterprise problem.
author: "Mike Tobias"
date: 2019-04-11
# linktitle: Test Post
# weight: 1
categories:
- AWS
series:
- Case Study: Netflix Voting Service
aliases:
- /blog/test/
---

This is part 1 of the series.  Feel free to skip around to other sections using the links below.

1. [Case Study and Grooming]({{< ref "/posts/aws-sqs-microservice-pipeline.md" >}})
2. [Simple API Endpoints with Serverless and Lambda]({{< ref "/posts/simple-api-endpoints-with-serverless-and-lambda.md" >}})
3. [Handling SQS Messages with Serverless Functions]({{< ref "/posts/handling-sqs-messages-with-serverless.md" >}})

---

#### Case Study

We are working at Netflix.  Yep, you and me both.  Our company, Netflix, has recently implemented their voting functionality where users can vote on different aspects of the show or movie they just watched.  We have been tasked with creating a pipeline which can receive and process an extremely large number of these votes in the most efficient, scalable, and cost effective way possible.  

#### Grooming

In this scenario, we can assume that the data will be passed from the front end as JSON.  The JSON will have a particular structure and we will have to provide a URL endpoint where the JSON can be sent.  Since the quantity of votes received per second or minute may change dramatically based on time of day or any number of other factors, we likely do not want to have a powerful server running all the time waiting to collect these votes.  This seems like an ideal situation for an [AWS API Gateway](<https://aws.amazon.com/api-gateway/>) connected to 1 or more serverless [Lambda](<https://aws.amazon.com/lambda/>) functions.  

We could probably use just 1 Lambda function to receive the data, verify and sanitize it, and then post it to our database, but that seems like a little bit much for just 1 little function.  We should probably have more than 1 function, each with a very specific job.  For this scenario, let's go with 1 function to receive the data, 1 to process the data, and 1 to add it to our database.

Since we may end up with an extremely high volume of calls to our API - think thousands of requests per second - it would be a good idea for us to "decouple" our pipeline.  We'll achieve this by using [AWS SQS](<https://aws.amazon.com/sqs/>) as a message queue service in between our very first "receiver" function and the rest of our serverless pipeline.  SQS will allow us to store a queue of messages which contain the data submitted through the API endpoint.  Then, later on, we can ask this queue for messages to process.  The key here is that we don't have to _immediately_ process and store the data from the API request.  We can add it to the queue, then process items from the queue in a more leisurely fashion.  This "decoupling" of our pipeline will help reduce the stress on our database and also helps keep our pipeline scalable and manageable.  

For this scenario, we will use [Dynamo DB](<https://aws.amazon.com/dynamodb/>) because it is flexible, cost-efficient, and highly scalable. It also has the added benefit of very smooth interoperability with Lambda and other AWS services. 

So, in summary, our project should include:

- An API endpoint (1 or more) for a front end to send vote data to.
- A Lambda function which will collect the data from this API endpoint and add it to...
- …an SQS message queue.
- 1 or more Lambda functions which will periodically ask the SQS message queue if there are any messages to process and, if so, process these messages (votes) and add them to...
- …a Dynamo DB table.

Let's get started!  The [next post]({{< ref "/posts/simple-api-endpoints-with-serverless-and-lambda.md" >}}) in this series will go over the first segment of our pipeline: the API gateway and the initial Lambda function.

