---
title: Simple API Endpoints with Serverless and Lambda
description: Using AWS Lambda and the Serverless Framework to build platform-agnostic API's and serverless functions.
author: "Mike Tobias"
date: 2019-04-16
# linktitle: Test Post
# weight: 1
categories:
- AWS
series:
- Case Study: Netflix Voting Service
---

This is part 2 of the series.  Feel free to skip around to other sections using the links below.

1. [Case Study and Grooming]({{< ref "AWS SQS Microservice Pipeline.md" >}})
2. [Simple API Endpoints with Serverless and Lambda]({{< ref "Simple API Endpoints with Serverless and Lambda.md" >}})
3. [Handling SQS Messages with Serverless Functions]({{< ref "Handling SQS Messages with Serverless.md" >}})

---

## Objective

Our objective in this segment is to create an API endpoint attached to a Lambda function which will handle vote submissions, adding each vote to an SQS Message Queue to be processed at a later time.  This is the first part of our voting pipeline, an application that we're designing as part of a fictitious case study.  See [part 1]() of this series for more info on that.  

## Intro

We'll be using the [Serverless Framework](<https://serverless.com/framework/>) to build our "back-end" functions and API endpoints.  This framework is platform-agnostic, meaning we don't _have_ to use AWS as our cloud platform.  We could use other providers as well!  The serverless functions are also a great choice here because they are efficient, cost-effective, and extremely scalable.  

Also, if you're unfamiliar with any of the subjects discussed in this post, I'd very much encourage you to read through some of the online documentation available for them.  Maybe do some googling or run through some of the "Getting Started" sections.  There's quite a bit of information out there, but here's some links that might be helpful:

- [Lambda Developer Docs](<https://docs.aws.amazon.com/lambda/latest/dg/welcome.html>)
- [Building Lambda functions with Nodejs](<https://docs.aws.amazon.com/lambda/latest/dg/programming-model.html>)
- [The AWS SDK for Nodejs](<https://aws.amazon.com/sdk-for-node-js/>)
- [Amazon Simple Queue Service (SQS)](<https://aws.amazon.com/sqs/>)
- [The Serverless Framework (Docs)](<https://serverless.com/framework/docs/>)

## Serverless Framework - Getting Started

First, we'll install the Serverless Framework.  There's some great directions on how you can do that [here](<https://serverless.com/framework/docs/getting-started/>), but essentially all you'll be doing is running `npm install -g serverless`.  This will install the cli tools for the framework, accessible through the `sls` or `serverless` commands.

Now we can initialize a new project directory and create our serverless functions.  First, create a root project directory for yourself.  Mine will be `~/projects/voting-project/`. I also chose to add a subdirectory in this root folder called `serverless-functions`, just to keep the serverless portion of this service separated.  I'll use this location to store all of my Lambda functions for the project.  So, let's `cd` into this directory and run the below command.

`sls create —template aws-nodejs —path voting-service`

This command will create all of the necessary files for your serverless function, using a Node.js template.  Let's open the `handler.js` file (full path: `~/projects/voting-project/serverless-functions/voting-service/handler.js`). This file contains some boilerplate code for a Node.js lambda function:

```javascript
// handler.js
'use strict';

module.exports.hello = async (event) => {
  return {
    statusCode: 200,
    body: JSON.stringify({
      message: 'Go Serverless v1.0! Your function executed successfully!',
      input: event,
    }),
  };

  // Use this code if you don't use the http event with the LAMBDA-PROXY integration
  // return { message: 'Go Serverless v1.0! Your function executed successfully!', event };
};
```

We can deploy this function immediately, if we wanted to!  Just `cd` into this directory - the `…/voting-service/` directory - and run `sls deploy`.  This will create a CloudFormation template and run it in AWS, which you can see in the AWS Management Console.  Then, to get rid of _everything_ that this deploy command did, you can either run `sls remove` or you can open the CloudFormation stack in AWS and delete the stack manually.

This intro template from `serverless` would only be this single Lambda function and nothing else.  What we need is a function that handles data and adds it to a SQS queue and an API endpoint to send our voting data to.  Let's add those things now.

## Add an API Endpoint

The main benefit of using the Serverless Framework comes from the `serverless.yml` file.  This file defines not only your serverless functions, but any other services and rules related to these functions.  There is quite a bit of documentation on this file (and other parts of the framework), which you can find [here](<https://serverless.com/framework/docs/>). To add an API endpoint, all it takes is to update our `serverless.yml` file to contain the right bits of information.  Here's the final result:

```yaml
# serverless.yml
service: voting-service

provider:
  name: aws
  region: us-east-1
  runtime: nodejs8.10
  stage: dev


functions:
  post-vote:
    handler: post-vote.postVote
    events:
      - http:
          path: vote
          method: post
```

In short, what we do here is:

- Define our service name (at the top)
- Define the provider information we'd like to use.  In our case, we're using AWS in the us-east-1 region, the runtime for our function will be nodejs, and we'd like to set our deployment stage as `dev` for the moment. 
- Define our functions.  We set the first function name to `post-vote` and point out that the handler (the actual function) can be found in the `post-vote` file at the `postVote` function (`<file>.<function name>`). Yes, these names are currently wrong.  We'll update them. 
- **Define the Events associated with this function**.  This is where we set up our API endpoint.  You can also set up many other types of events here which will trigger your Lambda function.  Check out all the event options [here](<https://serverless.com/framework/docs/providers/aws/events/>).
  - We define the endpoint as HTTP POST, with a url path of `/vote/`

You'll notice that the names of the file and function in our `.yml` file don't match up at the moment.  The `postVote` function does not currently exist, and neither does the `post-vote` file.  Currently, the `serverless.yml` file would have to show `handler.hello` for our function to work properly.  Let's update that `.js` file now before we test and deploy this API endpoint.  We'll change the names of the file and function and update the code within.

## Building our Serverless Function

First, a quick update of the file name.  We'll name the file `post-vote.js` and update the function name to `postVote`.

```javascript
// hanlder.js --> post-vote.js
'use strict';

module.exports.postVote = async (event) => {
...
```

Ok, now let's throw some useful code in this file.  Currently, our function simply accepts the `event` object provided by the API endpoint and returns an object with a `200` status code and a `body` property containing a simple message and the `event` object.  Let's update this so that, instead, we collect our voting data from the `event` parameter and add this voting data to a specific SQS queue.

```javascript
"use strict";
const AWS = require("aws-sdk");
const parseEventData = (apiEventData) => {

  // data should be passed in through query string params
  if (apiEventData.queryStringParameters) {
    return apiEventData.queryStringParameters;
  }

  return null;
};

AWS.config.update({region: "us-east-1"});

module.exports.postVote = async (event, context) => {
  const sqs = new AWS.SQS({apiVersion: "2012-11-05"});
  const sns = new AWS.SNS({apiVersion: "2010-03-31"});
  let voteQueueUrl;
  let voteData;

  voteData = parseEventData(event);

  if (!voteData) {
    return {
      statusCode: 500,
      isBase64Encoded: false,
      headers: {
        "Content-Type": "text/plain"
      },
      body: JSON.stringify({
        awsRequestId: context.awsRequestId,
        error: {
          message: "No query string parameters were found!"
        },
        input: event
      })
    };

  }
  
  try {
    voteQueueUrl = await sqs.getQueueUrl({
      QueueName: process.env.SQS_QUEUE_NAME,
    }).promise();
  
    await sqs.sendMessage({
      MessageBody: JSON.stringify(voteData),
      QueueUrl: voteQueueUrl.QueueUrl
    }).promise();
    console.log("SQS message sent successfully!");
  } catch (err) {
    console.error(`SQS message send FAILED. Error: ${err}`);
  }

  // notify our second function that there's a new vote to handle
  try {
    await sns.publish({
      Message: "New Vote Posted!",
      TopicArn: process.env.SNS_TOPIC_ARN
    });
    console.log("SNS published successfully!");
  } catch (err) {
    console.error(`SQS message send FAILED. Error: ${err}`);
  }


  return {
    statusCode: 200,
    headers: {
      "Content-Type": "text/plain"
    },
    isBase64Encoded: false,
    body: JSON.stringify({
      awsRequestId: context.awsRequestId,
      message: "Vote successfully processed.",
      input: event
    })
  };
};

```

A couple of things to go over here.  First, the AWS SDK is made available in Lambda to all supported languages.  In Nodejs, this is the `aws-sdk` package.  This package allows us to access other AWS services, in this case the SQS service.  Also, when using the AWS SDK, we always want to specify the region we're working in as well as a specific API version for the service that we use.  We've done that here near the top of the file.  Let's go over the code in a bit more detail.

First, we require the `aws-sdk` and establish a function for parsing the event data (provided by the API endpoint).  Our endpoint will be set up with "Lambda Proxy" enabled by default, which means the `event` object passed in to our function will have quite a bit of information in it.  This also means that our response from this function needs to have a `body` which is formatted properly.  We'll just use the `JSON.stringify()` method for this. 

We then set up a quick function for processing the data from our `event` object.  In our case, all we want to do is make sure that `event` _is_ an object and that it contains the `queryStringParameters` property.  This is where our voting data should be found.  If these things aren't available, we'll just throw an error with a quick message explaining that the voting data was not passed in correctly to the API endpoint.  This error can be caught in our `async` function and returned to the client with an appropriate status code.  But if all goes well, we'll return the `queryStringParameters`. 

In our main function, `postVote`, we first initialize the SQS and `SNS` objects and initialize the variables we'll use in this function.  Then, we get our voting data from the `event` parameter, using the function we created.  If the no voting data was found, we'll return a `400` status code and the error as a property of `body`. If we do have data, we'll continue by collecting the URL of our SQS queue and then adding a new message to this queue.  This bit will be in a try / catch block so that, if we hit any problems here, we can log an error message with the details.  After this, we'll publish an SNS message.  This will serve as one of the triggers for our next lambda function.  This other function will be subscribed to this SNS topic and will be triggered whenever we publish to that topic.  Again, we surround the publishing of the SNS message in a try / catch so that we can log a specific error if something goes wrong with this part.  Finally, if we ran everything successfully in our main `postVote` function, we'll return a 200 status and a success message in the body.   

----

An important thing to note in our above code is that most functions in the AWS SDK will be asynchronous calls, and can be made into a Nodejs `Promise` by calling the [`.promise()`](<https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/Request.html#promise-property>) function on the [`AWS Request`](<https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/Request.html>) object (returned by `getQueueUrl` and `sendMessage` in our case).  This returns a `Promise` instead of using the callback, which allows us to use `async` and `await` to write cleaner code!

----

## Permissions

If you've worked in AWS before, you'll know that we need to set up some permissions here in order for these functions to run properly. Our Lambda function will have a couple of permissions by default - logging to CloudWatch, for example, which is done with the `console.log` statements - but it won't have the ability to do things in SQS or SNS unless we explicitly give those permissions to it.  This is done through IAM Roles.

With the Serverless framework, we can set up a role within our code.  This makes management of the service as a whole much easier.  To do this, we'll open up the `serverless.yml` file again and make some changes.  Here's the result:

```yaml
service: voting-app

provider:
  name: aws
  runtime: nodejs8.10
  stage: ${opt:stage, 'dev'}
  region: ${opt:region, 'us-east-1'}
  environment:
    SQS_QUEUE_NAME: ${self:service}-${self:provider.stage}-queue
    SNS_TOPIC: ${self:service}-${self:provider.stage}-topic
  iamRoleStatements:
    - Effect: Allow
      Action:
        - SQS:*
      Resource: {"Fn::Join" : ["", ["arn:aws:sqs:${self:provider.region}:", {"Ref":"AWS::AccountId"}, ":${self:provider.environment.SQS_QUEUE_NAME}" ] ] }
    - Effect: Allow
      Action:
        - SNS:*
      Resource: {"Fn::Join":["", ["arn:aws:sns:${self:provider.region}:", {"Ref":"AWS::AccountId"}, ":${self:provider.environment.SNS_TOPIC}"]]}

functions:
  post-vote:
    handler: post-vote.postVote
    events:
      - http:
          path: vote
          method: post
    environment:
      SNS_TOPIC_ARN: {"Fn::Join":["", ["arn:aws:sns:${self:provider.region}:", {"Ref":"AWS::AccountId"}, ":${self:provider.environment.SNS_TOPIC}"]]}

resources:
  Resources:
    VotesQueue:
      Type: 'AWS::SQS::Queue'
      Properties:
        QueueName: ${self:provider.environment.SQS_QUEUE_NAME}
```

The first thing you've probably noticed is a lot of these: `{}`.  What's going on here? Well, in the serverless yaml files you can use variables and functions!  Take a quick look [here](https://serverless.com/framework/docs/providers/aws/guide/variables/) to get a good idea on how variables work, and [here](https://theserverlessway.com/aws/cloudformation/template-functions/) for a nice blog post on the functions.  Using variables and functions allows us to avoid repeating ourselves - see the [DRY Policy](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) - and also helps keep our naming conventions all the same throughout our pipeline. 

The `iamRoleStatements` section is where we establish our IAM Roles.  These roles allow our Lambda functions to interact with SQS and SNS.  Here we've just said "Let our service access this specific resource and take any action on it" for both SQS and SNS. The resources in question are established at the `Resource` fields, where we build an AWS ARN which will specifically point to our SNS Topic and SQS Queue.

We've also added *environment variables* here for the SQS queue name and the SNS topic name.  These environment variables can be accessed at runtime by our functions!  You may have noticed in our function code above that we are collecting the SQS queue name and SNS topic name from `process.env`.  The variables we establish here in the `environment` section end up in that `process.env` object! 

The last important bit in this file is the `resources` section.  This section will create actual AWS resources for us when we run our serverless service!   Here we create a SQS Queue, collecting the Queue Name from our environment variable at the top of the file. 

## Finishing Up

That pretty much does it for this bit of our pipeline.  We can test our function locally by installing the AWS SDK into our local project with `npm i aws-sdk` and then running `sls invoke local —function post-vote —data <insert data string>`.  Here's an example that I ran locally:

![](/images/blog/2019-04-17_sls-invoke-local.png)

The returned string of JSON doesn't look pretty, but it does show that the function is working properly.  If you had trouble running this locally, you'll likely have to [configure your aws cli](<https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html>) so that the AWS SDK we're using can access your AWS resources.  

To deploy this to AWS, we run `sls deploy` within our `/serverless-functions/voting-service/` directory.  This will deploy everything that we need into AWS, permissions and all!  Then, if we want to remove all of these resources in AWS we can simply run `sls remove` in the same directory.  Simple. 

The next step in our project will be another serverless function which checks our SQS queue for messages and handles these messages appropriately.  We'll build this bit in [the next post of this series]({{< ref "Handling SQS Messages with Serverless.md" >}}).  See you there!