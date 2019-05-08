---
draft: true

title: Handling SQS Messages with Serverless Functions
description: Using Serverless functions to collect and handle messages from an AWS SQS queue.
author: "Mike Tobias"
date: 2019-04-18
# linktitle: Test Post
# weight: 1
categories:
- AWS
series:
- Case Study: Netflix Voting Service
---

This is part 3 of the series.  Feel free to skip around to other sections using the links below.

1. [Case Study and Grooming]({{< ref "AWS SQS Microservice Pipeline.md" >}})
2. [Simple API Endpoints with Serverless and Lambda]({{< ref "Simple API Endpoints with Serverless and Lambda.md" >}})
3. [Handling SQS Messages with Serverless Functions]({{< ref "Handling SQS Messages with Serverless.md" >}})

---

## Objective
In our [last post]({{< ref "Simple API Endpoints with Serverless and Lambda.md" >}}) we set up an API endpoint and a serverless function which will receive voting data and then add it to our SQS Queue.  Now, we need to build a service which will periodically check this SQS Queue to see if there are messages in it, then process these messages.  We will create another serverless function for this purpose which will collect messages from the SQS Queue and then add this vote to a Dynamo DB table.

