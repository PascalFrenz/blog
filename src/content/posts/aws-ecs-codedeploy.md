---
title: "AWS CodeDeploy for ECS Fargate deployments"
description: 'Deploy ECS Fargate applications via CodeDeploy'
pubDate: 'Aug 31 2024'
image: '/aws-ecs-codedeploy.jpg'
tags:
  - AWS
  - ECS
  - Fargate
  - CodeDeploy
  - DevOps
  - Deployment
---

Deploying applications is one of the most important tasks in software development.
After all, software that is not deployed is not usable. Given the importance of the topic,
a lot of thought has gone into the deployment process of applications.

Given that I work mostly with AWS, I wanted to share my experience in creating a deployment process 
for business-critical applications running on ECS Fargate by utilizing AWS CodeDeploy.

For this blog post, I also created a simple application that you can use if you want play around with AWS CodeDeploy.
It is a simple webserver (written with Bun!) that simulates an application answering HTTP requests.
You can find the source code here: https://github.com/PascalFrenz/aws-ecs-codedeploy-terraform

## Deployment vs. Release

Before we start, let's talk about the basics of deployments.
A deployment takes an artifact (for example a Docker image), puts it into a target environment and starts it.
The main goal of a deployment is to replace the old code by the new code without disrupting the availability of the application.

This is distinctly different from a release, which has to goal to make the new code available to users.
Releases are best managed by a separate process, independent of the deployment process, for example by using 
feature flags.

Now that we have a better understanding of the differences between deployment and release, let's have a look at
the requirements for a deployment process.

## Deployment Process Requirements

The goal of a deployment process is to put the new code into the target environment as quickly as possible,
while preventing any disruption of the application as best as possible.

There are a couple of methods to consider in order to achieve this goal:

1. Gradually shift traffic to the new version of the application.
2. Automatically detect failures and roll back to the previous version of the application if necessary.
3. Have Break-Glass mechanisms in place to be able to operate the service even in unexpected situations.

Let's have a look at each of these options in more detail.

### Gradually shift traffic to the new version of the application

Gradually shifting traffic is probably one of the most thought-of options to reduce the risk of a deployment.
The idea is simple enough: by gradually shifting real traffic to the new version of the application,
the risk of user impact is always limited to the amount of traffic shifted.
Additionally, this approach gives one the ability to observe the new version of the application in production and
stop the shift or even roll back to the previous version if deemed necessary.

The obvious downside of this approach is that it requires more time to shift traffic rather than to just
immediately put the new version into production.

To balance the upsides and downsides of this approach, some different strategies can be considered:

1. Shift traffic linearly to the new version of the application.
2. Shift traffic in a canary fashion to the new version of the application.

Linear traffic shifts are very easy to understand, as per interval X, a percentage of traffic Y is shifted
to the new version of the application.

Canary shifts are different in that there is one traffic shift by X percent, followed by a bake time and then,
assuming the application is healthy, the traffic is shifted by 100-X percent.

### Automatic Failure Detection and Rollback

A key to limiting the impact of a bad deployment is to first recognise that a new version of the application is not
working properly and then to get rid of the bad version as quickly as possible.

A mechanism that monitors the application based on given metrics and automatically triggers actions when a certain
threshold is reached can be used to achieve this goal.

Examples of such metrics are plentiful. Here are some of the most common ones:

- CPU usage
- Memory usage
- Error rate
- Response time
- Number of messages in a queue
- Number of errors in a log file

It is not uncommon to combine multiple metrics into a single metric that is used to determine the health of the application.
Ultimately though, it is up to the developer to decide which metrics are
best suited to determine the health of the application.

Once the metrics are found, any time a threshold of a metric is exceeded during a deployment, the deployment process
should be stoppend and a rollback should be initiated. That way it is guaranteed that the application is not put into
production with a bad version. Also, this should not require any manual intervention.

### Break-Glass Mechanisms

Both of the previous requirements describe what should be in place to handle deployments on the happy path of the
deployment process. However, there are some situations where the team that is responsible for the application
is facing a situation where they need to override the default configuration of the deployment process.

Here are some examples of such situations:

1. The current version of the application is returning 5xx responses due to a downstream service that is not available.
   The team needs to deploy a new version of the application that implements a fix for this issue. The deployment
   process would fail due to the automatic failure detection detecting the 5xx responses and the deployment would
   stop before it started.
2. A new version of the application implements a new feature that is known to consume more memory than the current
   version. Due to the automatic failure detection, the deployment process would fail due to the memory usage
   exceeding the threshold and the deployment would stop before it started.
3. A schema change in an asynchronous producer upstream of the application is causing the application to fail while
   consuming the messages. The team needs to deploy a new version of the application that implements a fix for this
   issue. The deployment process would fail due to the automatic failure detection detecting new messages in a dead letter
   queue and the deployment would stop before it started.

In all of these cases, the team needs to be able to override the default configuration of the deployment process
in order to heal the failing application.


## Where CodeDeploy fits in

AWS CodeDeploy is a service that implements all the above-mentioned requirements for deployments within AWS.
It supports different target runtime environments to deploy to, for example ECS, Lambda and EC2.

CodeDeploy can be considered an orchestrator of already existing infrastructure provided by AWS.
In the case of ECS, CodeDeploy requires and uses an Application Load Balancer (which sits in front of the ECS service) and
ECS task definitions (which are the artifacts) to deploy your application. It does so by first staring the 
new version of the application in a different target group of the load balancer and then gradually shifting traffic
from the old target group (blue) to the new target group (green).

CodeDeploy also integrates with AWS CloudWatch to monitor the health of the application and to trigger automatic
rollbacks in case any of the given alerts are triggered. This allows deployment setups where the application is
continuously monitored and rolled back in case of any issues.

And on top of all of that, CodeDeploy allows the user to override these configurations on a per-deployment basis.
This allows the team to implement break-glass mechanisms as they see fit, for example within their CI/CD pipeline.






