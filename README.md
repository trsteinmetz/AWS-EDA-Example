## Example of Event Driven Architecture

### Introduction

This repository contains everything needed to create an Event Driven Architecture (EDA) on AWS. All you need to do is deploy the CloudFormation template provided in the `/cf-template` folder.

EDA is an architecture pattern where there are one or more event buses ([`AWS::Events::EventBus`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-events-eventbus.html)) that receive events from external sources such as users with the AWS Management Console or mobile applications, other AWS Services, and third-party SaaS provider partners. These events' metadata are evaluated by rules ([`AWS::Events::Rule`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-events-rule.html)) and, in case of a match, the event is sent further downstream to be handled by other AWS Services. 

<p align="center">
  <img src="images\small-EDA.png" alt="Event Driven Architecture Diagram">
</p>
<p align="center">A simple EDA</p>

The diagram above shows a small EDA workload that executes a Lambda Function every time an S3:Put event happens on an S3 Bucket.

### The Architecture

The diagram below depicts the EDA workload committed on this repository:

![Architecture Diagram](/images/EDA-diagram.png)
<p align="center">Event Driven Architecture Diagram</p>

Using the AWS Management Console, the user sends an event to the Event bus (1). EventBridge checks the event’s metadata against the three rules created for this workload (2). All rules are evaluated simultaneously, and the event can match more than one rule.

If the event’s metadata matches the criteria in `rule-stateMachine`, it triggers the execution of an AWS StepFunction ([`AWS::StepFunctions::StateMachine`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-stepfunctions-statemachine.html)) (3). If the matched rule is `rule-api`, the event is sent to an API Endpoint ([`AWS::Serverless::Api`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigatewayv2-api.html)) (4). A Lambda Function is executed, and the result is logged in CloudWatch (5). Finally, if `rule-snsTopic` matches the event, it is sent to a SNS Topic ([`AWS::SNS::Topic`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-sns-topic.html)) (6). From there, the message is sent to an SQS Queue ([`AWS::SQS::Queue`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-sqs-queue.html)) subscribed to the Topic (7). The Queue has its dead-letter queue configured and, if an error occurs, the Topic's `LoggingConfig` property handles logging in CloudWatch (8).

### Usage

To deploy the [`CloudFormation`](https://aws.amazon.com/cloudformation/) [`stack`](cf-template\event-driven-architecture.yaml), follow these steps:

1. Clone this repository, running the command:

```bash
git clone https://github.com/trsteinmetz/AWS-EDA-Example
```
2. cd into the `cf-template` folder and run:

```bash
aws cloudformation create-stack --stack-name orders-eda --template-body file://event-driven-architecture.yaml --parameters ParameterKey=pApiConnectionPassword,ParameterValue=myPassword --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND
```

Quite a long command, right? It deserves some pointers:

- `ParameterKey` and `ParameterValue` are parameters been passed to CloudFormation. There is a Lambda Function that will perform a very simple authentication method: it compares `user` and `password` parameters against the hard-coded “myUser” and “myPassword” values. If those exact values are not passed, you will see an unauthorized access exception on CloudWatch logs. Deploying the stack from the CloudFormation page eliminates that risk. The parameters field are filled with the default values automatically.
- `CAPABILITY_IAM` grants CloudFormation permissions to IAM resources (Roles, Policies and Users) within the stack.
- `CAPABILITY_NAMED_IAM` grants CloudFormation permission to create named IAM resources (especially custom named roles) described within the stack.
- `CAPABILITY_AUTO_EXPAND` allows CloudFormations to run macros necessary to deploy some of the resources within the stack.

### Testing the Workload

After the stack is successfully deployed, open the AWS EventBridge page on the AWS Management Console. There is a link to the `orders` event bus created by the stack in the outputs tab. First, head over to the Rules page in the navigation menu. Let’s look at the event and rules configurations.

!["CloudFormation Stacks Page"](/images/outputs.png)

The JSON below describes the event metadata. The `source`, `detail.location`, and `detail.category` are the properties evaluated by the rules. In fact, when sending events via the Management Console, the remaining properties may be omitted.

```json
{
  "version": "0",
  "id": "83218614-0f1f-7060-ccae-162a04c20f25",
  "detail-type": "Event Example",
  "source": "com.aws.orders",
  "account": "738301555873",
  "time": "2024-02-09T16:41:54Z",
  "region": "us-east-1",
  "resources": [],
  "detail": {
    "location": "us-east",
    "category": "lab-supplies"
  }
}
```

Make sure that you selected the `orders` Event bus. The default event bus may appear selected because, well, its the **default** event bus. 

Click on each of the rules displayed on the Rules page. Take a look at the **event pattern** of each rule. These patterns are compared against the rules for any event we send to the orders event bus. You'll see that everything connects in just a little while.

Now head over to the “Event buses” page. Let’s trigger some rules.

#### The rule-api

The rule-api has only one criterion: the source property of the event must be equal to “com.aws.orders”.

Click on the rule to see its Event pattern.

```json
{
  "source": ["com.aws.orders"]
}
```

!["Triggering the API rule"](/images/rule-api.png)

This is a development rule: since all rules must have this field and value, every rule will also trigger the API. By doing so, we guarantee that every event will be logged on CloudWatch for debugging purposes.

The Event detail field can be filled with any type of JSON format data, as the exemple below.

```JSON
{
  "what": "this is just an example of Event detail payload.",
  "So?...": "You'll find this message in CloudWatch logs."
}
```

#### The rule-stateMachine

Here's the rule event pattern:

```json
{
  "source": ["com.aws.orders"],
  "detail": {
    "location": ["eu-east", "eu-west"]
  }
}
```

The rule-stateMachine evaluates the following conditions:

Source field value is equal to “com.aws.orders”, and
The detail.location property must have either “eu-east” or “eu-west” values.

!["Triggering the API rule"](/images/rule-stateMachine.png)

You can ascertain that the rule was triggered by heading over to the StepFunctions page, and from there to the State Machines on the navigation menu. Results of the state machine’s executions can be found there.

![ ](/images/sm-executions.png)

#### The rule-snsTopic

Look at the rule-snsTopic event pattern below:

```json
{
  "source": ["com.aws.orders"],
  "detail": {
    "location": ["us-east", "us-west"],
    "category": ["lab-supplies"]
  }
}
```

The rule-snsTopic evaluates the following conditions:

- Source field value is “com.aws.orders”,
- The detail.category property must be equal to “lab-supplies”, and
- The detail.location property must have either “us-east” or “us-west” values.

![ ](/images/rule-snsTopic.png)

You can verify that the rule was triggered by going to the SQS Queue of the stack. There should be messages there waiting to be polled.

![ ](/images/sqs-total-messages.png)

If an error occurs, the details (probably the SNS Topic not having the right permissions to send messages to the SQS Queue) can be found in the CloudWatch’s log group.

### Sending Multiple Events to the orders event bus

The [`/test-events`](/test-events/) folder has two files that help you send multiple events related to rule-stateMachine and rule-snsTopic rules. These events were used for testing purposes; therefore, not every event will match the rules.

Run the command below in the CLI to test against the `rule-stateMachine` rule:

```console
aws events put-events --entries file://test-events/rule-stateMachine-events.json
```

Run the following command in the CLI to test against the `rule-snsTopic` rule:

```console
aws events put-events --entries file://test-events/rule-snsTopic-events.json
```

### Conclusion

This repository provides a comprehensive example of implementing an Event Driven Architecture (EDA) on AWS using CloudFormation templates and AWS services like EventBridge, Lambda, Step Functions, SNS, and SQS. By following the instructions in this README, you can deploy the EDA workload and understand how events are processed through various rules and services in the architecture. The [`event-driven-architecture.yaml`](/cf-template/event-driven-architecture.yaml) file is a great starting point for developing large, complex EDA workloads to manage thousands of AWS Services.

Implementing an EDA can bring several benefits to your AWS infrastructure, including improved scalability, decoupling of components, and better handling of asynchronous communication between services. This example serves as a valuable resource for anyone looking to leverage event-driven patterns in their AWS projects.

Always tear down the applications after you finish your practice sessions. To do this, simply head over to the CloudFormation page, then stacks and click the `Delete` button.

### Contact Me

For any questions, feedback, or inquiries, feel free to reach out to me at tarcisio.roberto@gmail.com. I welcome any suggestions for improving this repository or discussing AWS architecture and best practices further.