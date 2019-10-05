---
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

1. [Case Study and Grooming]({{< ref "/posts/aws-sqs-microservice-pipeline.md" >}})
2. [Simple API Endpoints with Serverless and Lambda]({{< ref "/posts/simple-api-endpoints-with-serverless-and-lambda.md" >}})
3. [Handling SQS Messages with Serverless Functions]({{< ref "/posts/handling-sqs-messages-with-serverless.md" >}})

---

#### Objective
In our [last post]({{< ref "/posts/simple-api-endpoints-with-serverless-and-lambda.md" >}}) we set up an API endpoint and a serverless function which will receive voting data, add it to our SQS Queue, then publish to an SNS topic. Now, we need to build a function which will consume messages from this SQS Queue and add them to a database. We'll use DynamoDB for our database because it's scalable, reliable, NoSQL, and integrates well with our pipeline. 

#### The new function

Let's get right to it. Here's our second function: 

```javascript
"use strict";
const AWS = require("aws-sdk");
const sqs = new AWS.SQS({apiVersion: "2012-11-05"});
const db = new AWS.DynamoDB.DocumentClient({apiVersion: "2012-08-10"});

AWS.config.update({region: "us-east-1"});

const getQueueUrl = async () => {
  let queueUrlResponse = await sqs.getQueueUrl({
    QueueName: process.env.SQS_QUEUE_NAME
  }).promise();

  return queueUrlResponse.QueueUrl;
};

const getMessages = async (queueUrl) => {
  let messagesResponse = await sqs.receiveMessage({
    QueueUrl: queueUrl,
    WaitTimeSeconds: 5,
    MaxNumberOfMessages: 10
  }).promise();

  return messagesResponse && messagesResponse.Messages
    ? messagesResponse.Messages
    : [];
};

module.exports.handleSqsMessages = async (event) => {
  const dbTableName = process.env.DYNAMODB_TABLE;
  const sqsQueueName = process.env.SQS_QUEUE_NAME;
  const timestamp = new Date().getTime();

  if (event && event.Records) {
    console.log("Event Records: ", event.Records);
    let message = event.Records[0].Message;
    console.log("Event received.");
    console.log(`Event SNS Message: ${message}.`);
  }

  let canProcessMessages = true;
  let queueUrl = await getQueueUrl(sqsQueueName);

  if (!queueUrl) {
    canProcessMessages = false;
  }

  while (canProcessMessages) {
    let messages = await getMessages(queueUrl);

    if (!messages || !messages.length) {
      console.log(`No messages found in queue ${sqsQueueName}.`);
      canProcessMessages = false;
      return 0;
    }

    for (let msg of messages) {
      let msgId = msg.MessageId;
      let msgBody = JSON.parse(msg.Body);

      if (!(msgId && msgBody)) {
        // go to next message
        continue;
      }
      
      console.log(`processing message (id: ${msgId})`);

      try {
        await db.put({
          TableName: dbTableName,
          Item: {
            id: msgId,
            VoteData: msgBody,
            createdAt: timestamp,
            updatedAt: timestamp
          }
        }).promise();

        console.log(`DB PUT successful! Message Id: ${msgId}`);
      } catch (err) {
        console.error(`DB PUT failed! Message Id: ${msgId}. Error: ${err}`);
        return 1;
      }

      try {
        await sqs.deleteMessage({
          QueueUrl: queueUrl,
          ReceiptHandle: msg.ReceiptHandle
        }).promise();

        console.log(`Message deleted from SQS Queue. Queue URL: ${queueUrl} | Message Receipt Handle: ${msg.ReceiptHandle}`);
      } catch (err) {
        console.error(`Message deletion FAILED. Message Receipt Handle: ${msg.ReceiptHandle}. Error: ${err}`);
        return 1;
      }
    }
  }

  return 0;
};
"use strict";
const AWS = require("aws-sdk");
const sqs = new AWS.SQS({apiVersion: "2012-11-05"});
const db = new AWS.DynamoDB.DocumentClient({apiVersion: "2012-08-10"});

AWS.config.update({region: "us-east-1"});

const getQueueUrl = async () => {
  let queueUrlResponse = await sqs.getQueueUrl({
    QueueName: process.env.SQS_QUEUE_NAME
  }).promise();

  return queueUrlResponse.QueueUrl;
};

const getMessages = async (queueUrl) => {
  let messagesResponse = await sqs.receiveMessage({
    QueueUrl: queueUrl,
    WaitTimeSeconds: 5,
    MaxNumberOfMessages: 10
  }).promise();

  return messagesResponse && messagesResponse.Messages
    ? messagesResponse.Messages
    : [];
};

module.exports.handleSqsMessages = async (event) => {
  const dbTableName = process.env.DYNAMODB_TABLE;
  const sqsQueueName = process.env.SQS_QUEUE_NAME;
  const timestamp = new Date().getTime();

  if (event && event.Records) {
    console.log("Event Records: ", event.Records);
    let message = event.Records[0].Message;
    console.log("Event received.");
    console.log(`Event SNS Message: ${message}.`);
  }

  let canProcessMessages = true;
  let queueUrl = await getQueueUrl(sqsQueueName);

  if (!queueUrl) {
    canProcessMessages = false;
  }

  while (canProcessMessages) {
    let messages = await getMessages(queueUrl);

    if (!messages || !messages.length) {
      console.log(`No messages found in queue ${sqsQueueName}.`);
      canProcessMessages = false;
      return 0;
    }

    for (let msg of messages) {
      let msgId = msg.MessageId;
      let msgBody = JSON.parse(msg.Body);

      if (!(msgId && msgBody)) {
        // go to next message
        continue;
      }
      
      console.log(`processing message (id: ${msgId})`);

      try {
        await db.put({
          TableName: dbTableName,
          Item: {
            id: msgId,
            VoteData: msgBody,
            createdAt: timestamp,
            updatedAt: timestamp
          }
        }).promise();

        console.log(`DB PUT successful! Message Id: ${msgId}`);
      } catch (err) {
        console.error(`DB PUT failed! Message Id: ${msgId}. Error: ${err}`);
        return 1;
      }

      try {
        await sqs.deleteMessage({
          QueueUrl: queueUrl,
          ReceiptHandle: msg.ReceiptHandle
        }).promise();

        console.log(`Message deleted from SQS Queue. Queue URL: ${queueUrl} | Message Receipt Handle: ${msg.ReceiptHandle}`);
      } catch (err) {
        console.error(`Message deletion FAILED. Message Receipt Handle: ${msg.ReceiptHandle}. Error: ${err}`);
        return 1;
      }
    }
  }

  return 0;
};

```

So what are we working with here? We've got a couple "helper" functions at the top for getting the SQS queue URL and retrieving messages from this queue. Then, we've got the main function. In there, we're really only doing a few things: logging the event that triggered this function (using `console.log`), collecting messages from the SQS queue, adding all messages found to our database, then deleting all the messages we've processed from the SQS queue. Let's go over it a little more in detail.

First we first collect our databse table name and SQS queue name from our environment variables. You'll notice the `DYNAMODB_TABLE` environment variable is a new one. We've added that to the `serverless.yml` file, which well see in a moment. We also establish a timestamp which we can use later on in the function.

Next, we log the event `Records` if any are found. This is only used for tracking purposes. By adding a few of these `console.log` statements throughout our function, we are then able to look into our logs in AWS CloudWatch to see a detailed accounting of what happened when this function ran.  The `Records` object will exist only if the function was triggered by an SNS publish.

Now we collect our SQS queue URL and begin processing messages. Assuming a queue URL was returned, we'll establish a loop which only closes when we decide we can no longer process messages (`canProcessMessages=false`). In the loop, we grab the messages using our `getMessages` function. If this returns no messages, we stop the loop and the function ends.  However, if it returns some messages we process each one of them. This means collecting the `MessageId` and the message `Body`, then running a `PUT` to our dynamoDB table. The format of the object we pass to this `db.put` is important. You can find more info about this formatting in the [nodejs aws-sdk docs](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB/DocumentClient.html#put-property). 

If any errors were hit on our way to this point, we've logged them to AWS CloudWatch and returned a non-zero value, ending the function.  But, if we made it this far without errors, we delete the message from our SQS queue and then we're done!

#### The updated yaml file
So, now comes the second major part of this segment of our pipeline: the `serverless.yml` file. Here's the updated version. Take a quick read.

```yaml
service: voting-app

provider:
  name: aws
  runtime: nodejs8.10
  stage: ${opt:stage, 'dev'}
  region: ${opt:region, 'us-east-1'}
  environment:
    DYNAMODB_TABLE: ${file(../voting-db/serverless.yml):service}-${self:provider.stage}-table
    SQS_QUEUE_NAME: ${self:service}-${self:provider.stage}-queue
    SNS_TOPIC: ${self:service}-${self:provider.stage}-topic
  iamRoleStatements:
    - Effect: Allow
      Action:
        - SQS:*
      Resource: {"Fn::Join" : ["", ["arn:aws:sqs:${self:provider.region}:", {"Ref":"AWS::AccountId"}, ":${self:provider.environment.SQS_QUEUE_NAME}" ] ] }
    - Effect: Allow
      Action:
        - dynamoDB:*
      Resource: {"Fn::Join" : ["", ["arn:aws:dynamodb:${self:provider.region}:", {"Ref":"AWS::AccountId"}, ":table/${self:provider.environment.DYNAMODB_TABLE}" ] ] }
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
  handle-votes:
    handler: handle-votes.handleSqsMessages
    events:
      - schedule: rate(1 minute)
      - sns: ${self:provider.environment.SNS_TOPIC}

resources:
  Resources:
    VotesQueue:
      Type: 'AWS::SQS::Queue'
      Properties:
        QueueName: ${self:provider.environment.SQS_QUEUE_NAME}
```

The new additions to point out here are not many. We've added the new environment variable, `DYNAMODB_TABLE`. We added a new `iamRoleStatement` which allows our functions access to dynamoDB.  And we added a new `handle-votes` section in our `functions`. This is the interesting bit!  This function has 2 `events` set up for it.  One of these events is a `schedule` which will trigger the function every "1 minute".  This is probably excessive, and could be increased to a larger amount of time in production.  The idea with this schedule is just to have the function run some "cleanup" every now and then to make sure that all of our SQS messages get handled.  However, we should be able to rely on the next event to do most of the work here.  The `sns` event will subscribe our `handle-votes` function to a specific SNS Topic, which we've provided by referencing an environment variable in this file.  Since the function is subscribed, it will be triggered when our _first_ function (`post-vote`) publishes a message to this topic. 

That's all for this file!  But what about DynamoDB?  We aren't creating that resource here, are we?  Correct!  The reason we don't include databases here is that, in a production environment, databases usually exist already or at least won't be changed very often.  This serverless file is designed so that you can create, update, and destroy all of these resources and services at will. This isn't the approach you'd want to take with your data, however.

If you did want to create a Dynamo database and table using serverless, maybe for testing purposes, you can still do that!  Serverless has the power!  The best plan here would be to create a new `serverless.yml` file somewhere **outside** of your `voting-functions` directory.  The idea is to have a different area where you can run your `serverless` command in the terminal, so that when you update your Lambda functions and other resources you won't also try to re-deploy your DynamoDB table when you update these functions using `serverless deploy`.  So, what would that new file look like for creating your database?  Here you go:

```yaml
service: voting-db

custom:
  tableName: ${self:service}-${self:provider.stage}-table

provider:
  name: aws
  runtime: nodejs8.10
  stage: ${opt:stage, 'dev'}
  region: ${opt:region, 'us-east-1'}

resources:
  Resources:
    VotesDynamoDbTable:
      Type: 'AWS::DynamoDB::Table'
      DeletionPolicy: Retain
      Properties:
        AttributeDefinitions:
          - AttributeName: id
            AttributeType: S
        KeySchema:
          - AttributeName: id
            KeyType: HASH
        ProvisionedThroughput:
          ReadCapacityUnits: 1
          WriteCapacityUnits: 1
        TableName: ${self:custom.tableName}
```

A few sections here probably look familiar to you.  The resources section creates our resources and in it we've established our `VotesDynamoDbTable`.  Tables in DynamoDB require at least 1 `Key` to serve as the primary key.  This is created by first establishing an `AttributeDefinition` and then creating our `KeySchema` which references the name of our newly created Attribute.  We then provide the `TableName` at the end.  Check out the [serverless examples repo](https://github.com/serverless/examples/blob/master/aws-node-rest-api-with-dynamodb/serverless.yml) for more examples of using dynamodb with the serverless framework.

We also use a new section here called `custom`.  This is a good area for you to define local variables for your file. Since this new serverless service doesn't need to use the dynamoDB table name anywhere else, we can just put it in this `custom` section for reference within this file.

Now, you can run `serverless deploy` in this directory to create this DynamoDB table!

#### Other thoughts

One thought you may have had while reading this is:

> What if we fail to delete the SQS message after processing it? Wouldn't we end up adding a duplicate to the database later on?

No. The `MessageId` value provided by each message from our SQS queue will remain the same if we fail to delete that item. We will, indeed, end up collecting this message again later on, but since we're using this value as our primary key in the DynamoDB table we'd avoid duplicating it.  When you try to put a record in a DynamoDB table and the record's primary key already exists, DyanmoDB will wimply simply [overwrite the old item with the new](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_PutItem.html) one. Since the data from our SQS message will not have changed, this would be alright with us!

#### Finishing Up

We have now successfully built our pipeline!  To sum it all up, we now have:
* An API endpoint where a client can submit voting data.
* A serverless Lambda function which will accept data from this endpoint, add it to a SQS queue, and publish an SNS message.
* Another serverless Lambda function which will trigger when a message is published to this SNS topic OR on a timed schedule. This function will collect votes from the SQS queue, add them to a DynamoDB table, and then delete them from the queue.
* A DynamoDB table to store the voting data.

This pipeline is extremely scalable and can be created, updated, and deleted at the push of a button! You can try it now by cloning [this repo](https://github.com/mct-dev/voting-app) and running `npm deploy`. To remove all of these resources from AWS, just run `npm run remove`.  Keep in mind that you'll need to be authenticated to your AWS account for any of this to work.  You can find out how to do this [here](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html).

One important note: if you're creating your database with the serverless template I've shown above, you won't be able to remove it with a `serverless remove` command.  This is for all the same "data integrity" and safety reasons as we mentioned before.  AWS won't let you destroy database things without you explicitly telling it to do so.

And with that, I think that'll do it for now.  If you're still reading, I appreciate you sticking it out.  Be sure to check in for the next one.  I'll make sure we build something even cooler. :)