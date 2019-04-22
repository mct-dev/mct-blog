---
# draft: true

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

# Simple API Endpoints with Serverless and Lambda

---

This is part 1 of the series.  Feel free to skip around to other sections using the links below.

1. [Case Study and Grooming](#) 
2. [Simple API Endpoints with Serverless and Lambda](#)

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

Now we can initialize a new project directory and create our serverless functions.  First, create a root project directory for yourself.  Mine will be `~/projects/voting-service/`. I also chose to add a subdirectory in this root folder called `serverless-functions`, just to keep the serverless portion of this service separated.  I'll use this location to store all of my Lambda functions for the project.  So, let's `cd` into this directory and run the below command.

`sls create —template aws-nodejs —path voting-service`

This command will create all of the necessary files for your serverless function, using a Node.js template.  Let's open the `handler.js` file (full path: `~/projects/voting-service/serverless-functions/voting-service/handler.js`). This file contains some boilerplate code for a Node.js lambda function:

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
// post-vote.js
'use strict';
const AWS = require('aws-sdk');

AWS.config.update({region: 'us-east-1'});

module.exports.postVote = async (event, context) => {
  let sqs = new AWS.SQS({apiVersion: '2012-11-05'});
  let voteQueueUrl;
  let sqsMessageResponse;

  try {
    voteQueueUrl = await sqs.getQueueUrl({
        QueueName: 'voting-app-queue',
    }).promise();
    sqsMessageResponse = await sqs.sendMessage({
      MessageBody: JSON.stringify(event),
      QueueUrl: voteQueueUrl.QueueUrl
    }).promise()
  }
  catch (err) {
    return {
      statusCode: 500,
      body: JSON.stringify(err)
    }
  }
  return {
    statusCode: 200,
    body: JSON.stringify({
      voteQueueUrl,
      sqsMessageResponse,
      input: event,
    }),
  };
};
```

A couple of things to go over here.  First, the AWS SDK is made available in Lambda to any programming language that needs it.  In Nodejs, this is the `aws-sdk` package.  This package allows us to access other AWS services, in this case the SQS service.  Also, when using the AWS SDK, we always want to specify the region we're working in as well as a specific API version for the service that we use.  We've done that here near the top of the file.  Let's go over the code in a bit more detail.

We first create a new `AWS.SQS` object, specifying its most recent API version.  Then, we collect the `QueueUrl` for a SQS Queue named `voting-app-queue`.  This is a SQS queue that I've created directly through the AWS management console.  You can use the AWS SDK to create a queue, but we don't necessarily need to programmatically create a queue in this scenario, so I just created one by hand.  Once we have this URL, we can add messages to our queue!  So, the next step (`sqs.sendMessage`) sends a message to our QueueUrl adding the `event` as the MessageBody.  

Another cool thing to point out here is the `async` / `await` syntax.  Lambda supports more recent versions of Nodejs, which allows us to use this syntax to handle promises.  This makes our code much cleaner, avoiding lots of callbacks and such, especially while dealing with the AWS SDK - most things in there will be asynchronous calls, and can be made into a Nodejs `Promise` by calling the [`.promise()`](<https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/Request.html#promise-property>) function on the [`AWS Request`](<https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/Request.html>) object (returned by `getQueueUrl` and `sendMessage` in our case). 

This function is also associated with an API endpoint, so we want to make sure we return something to that API call.  In this case, if something goes wrong while dealing with our SQS service, we'll return a `500` error code ("Internal Server Error") as well as the error details.  We could probably also add some more checks in this function to verify the data, returning different error codes if needed, but for now this will do just fine.  If all goes well in our function, we'll have successfully added a message to the SQS queue and we return a `200` status and some information about our queue and the message we sent.

## Permissions

If you've worked in AWS before, you'll know that we need to set up some permissions here. Our Lambda function will have a couple of permissions by default - logging to CloudWatch, for example - but it won't have the ability to do things in SQS unless we explicitly give those permissions to it.  This is done through "Roles" in AWS.  

With the Serverless framework, we can set up a role within our code.  This makes management of the service as a whole much easier.  To do this, we'll open up the `serverless.yml` file again and make a minor change.  Here's the result:

```yaml
# serverless.yml
service: voting-app

provider:
  name: aws
  runtime: nodejs8.10
  stage: dev
  region: us-east-1
  iamRoleStatements:
    - Effect: "Allow"
      Action:
        - sqs:*
      Resource: "*"

functions:
  post-vote:
    handler: post-vote.postVote
    events:
      - http:
          path: vote
          method: post
```

With the `iamRoleStatements` section, we are able to define the AWS Roles that we want our functions to assume.  Here we've added a role which allows our function to take *any* action on *any* SQS resource.  You can check [here](<https://serverless.com/framework/docs/providers/aws/guide/iam/>) for more information on how to set up roles in the Serverless Framework.

## Finishing Up

That pretty much does it for this bit of our pipeline.  We can test our function locally by installing the AWS SDK into our local project with `npm i aws-sdk` and then running `sls invoke local —function post-vote —data <insert data string>`.  Here's an example that I ran locally:

![](/images/blog/2019-04-17_sls-invoke-local.png)

The returned string of JSON doesn't look pretty, but it does show that the function is working properly.  If you had trouble running this locally, you'll likely have to [configure your aws cli](<https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html>) so that the AWS SDK we're using can access your AWS resources.  

We can now run `sls deploy` within our `/serverless-functions/voting-service/` directory and it will deploy everything that we need into AWS, permissions and all!  Then, if we want to remove all of these resources in AWS we can simply run `sls remove` in the same directory.  Simple. 

The next step in our service will be another serverless function which checks our SQS queue for messages and handles these messages appropriately.  We'll build this bit in [the next post of this series](#).  See you there!