---
title: "Merging partial logs with FluentBit on AWS ECS Fargate"
description: 'How to merge partial logs produced by the container runtime of AWS Fargate'
pubDate: 'Mar 17 2024'
image: '/merge_logs_preview_image.jpeg'
tags:
  - AWS
  - ECS
  - Fargate
  - FluentBit
  - Datadog
  - Logging
---

Handling logging in an application is not as easy as one might think. Especially if logs need to be ingested into other
systems for easy analysis or storage purposes, things can get complicated quickly.

At work, we faced one such problem. We are using Datadog as our logging solution and noticed that some log messages
were ingested not as one entry, but as many partial messages. Since logs are printed as JSON one-liners to console for easier parsing,
having partial log messages ingested messes up the handling of the logs after ingestion.

<details>
<summary>Example log message</summary>

Here is an example of an ordinary log message (pretty printed here, would be just one line):

```json
{
  "level": "info",
  "logger": "org.example.company.Logger",
  "message": "This is a log message",
  "context": {
    "dd": {
      "env": "prod",
      "service": "simple-service",
      "tags": "..."
    }
  },
  "...": "..."
}
```

</details>

A quick analysis did not reveal any obvious cause of the splitting of messages. Datadog handles ingestion via their
agent when running containerized workloads and within that agent, logs are not split if < 1MB in size (which was the case here).

The culprit must have been somewhere else but as it turned out, it's not that easy...

## Docker

Our applications are packaged as Docker Images and run on AWS ECS Fargate. This means there is an underlying container
runtime, which manages the handling of logs on stdout or stderr to the host system. When analyzing problems with
log messages, this might not be an obvious place to look for root causes.
In our case however, the container runtime used by Fargate split our messages when they exceeded ***16k*** characters in length.
When logging java exceptions, this limit can be reached quickly.

<details>
<summary>Example of a split log message</summary>

```json
{
  "partial_message": true,
  "partial_first": true,
  "message": "{\"level\":\"info\",\"logger\":\"org.example.company.Logger\",\"message\":\"This is a log messa",
  "partial_id": "i123o561a24suf457zov2b534iaufg16hz7u45v2btafg",
  "partial_index": "1"
}
```
```json
{
  "partial_message": true,
  "partial_first": false,
  "message": "ge\",\"context\":{\"dd\":{\"env\":\"prod\",\"service\":\"simple-service\",\"tags\":\"...\"}},\"...\":\"...\"}",
  "partial_id": "i123o561a24suf457zov2b534iaufg16hz7u45v2btafg",
  "partial_index": "2"
}
```

</details>

Luckily, another Log Driver - FluentBit - can recombine these split logs. This does imply some configuration from your
side though.
You have to either create your own derivative of the official `aws-for-fluent-bit` docker image
or use their `init`-variation of the official image together with custom configuration via files on S3.

