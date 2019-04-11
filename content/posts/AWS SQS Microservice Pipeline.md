---
draft: true

title: A Serverless Pipeline with AWS
description: Building a voting pipeline using AWS services
author: "Mike Tobias"
date: 2019-04-11
# linktitle: Test Post
# weight: 1
categories:
- AWS
series:
- Case Study: Netflix Voting Pipeline
aliases:
- /blog/test/
---

# AWS SQS Microservice Pipeline

---

This is part 1 of the series.  Feel free to skip around to other sections using the links below.

[Part 1: Case Study and Grooming]() 
[Part 2: Getting Started]()



---

## Case Study

We are working at Netflix.  Yep, you and me both.  Our company, Netflix, has recently implemented their voting functionality where users can vote on different aspects of the show or movie they just watched.  We have been tasked with creating a pipeline which can receive and process an extremely large number of these votes in the most efficient, scalable, and cost effective way possible.  

## Grooming the objective

In this scenario, we can assume that the data will be passed from the front end as JSON.  The JSON will have a particular structure and we will have to provide a URL endpoint where the JSON can be sent.  Since the quantity of votes received per second or minute may change dramatically based on time of day or any number of other factors, we likely do not want to have a powerful server running all the time waiting to collect these votes.  This seems like an ideal situation for an [AWS API Gateway](<https://aws.amazon.com/api-gateway/>) connected to 1 or more serverless [Lambda](<https://aws.amazon.com/lambda/>) functions.  

So, we are now sending lots of JSON voting data to an API endpoint.  Each time we send JSON data, a Lambda function is triggered and given this data.  But what does this function do with the data?  Our objective is to store this data in a database but, if our function sent the data directly to the database, it's possible that we could have millions of functions all trying to write to the database at nearly the same time.  This could cause some issues.  So, keeping in mind that we are aiming to build a *scalable* solution, we could consider [AWS SQS](<https://aws.amazon.com/sqs/>) for this portion of our pipeline.  Simple Queue Service (SQS) would allow use to create and store a queue of tasks which we could then pull from later at our leisure. Similar to AWS Lambda, SQS scales elastically based on the demand that you place on it and your costs are only based on usage, so our pipeline stays scalable and cost effective. 

