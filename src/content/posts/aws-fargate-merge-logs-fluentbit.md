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

So I went searching for the part of the application that split up the logs. Little did I know how many moving parts
were involved...

## Docker

Our applications are packaged as Docker Images and run on AWS ECS Fargate. This means there is an underlying container
runtime, which manages the handling of logs on stdout or stderr to the host system. When analyzing problems with
log messages, this might not be an obvious place to look for the root cause. Everyone just assumes that logs printed by
the application will be passed to the host by the container runtime.
It's not that simple though. Container runtimes use an abstraction layer for logging, so-called [logging drivers](https://docs.docker.com/config/containers/logging/configure/).

In our case, the logging driver of the container runtime used by Fargate split our messages when they exceeded ***16k*** characters in length.
When logging Java stack traces, this limit can be reached rather quickly.

Here is an example of what these log entries looked like:

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

Luckily, another Log Driver called __FluentBit__ can recombine these split logs. This does imply some configuration from your
side though.
You have to either use their [`init`-variation](https://github.com/aws/aws-for-fluent-bit?tab=readme-ov-file#using-the-init-tag)
of the official image together with custom configuration via files on S3 or create your own derivative of the official `aws-for-fluent-bit` docker image.

We chose the latter and built our own docker image derivative and open-sourced it, so you don't have to built it yourself!
Here is the link to the GitHub repository: https://github.com/Hapag-Lloyd/fluent-bit-multiline

One deployment later and logs started to pour in to our system again. This time the formatting looked correct, but
we received lots and lots of duplicate log entries... The logs were split once again!
Their only distinguishing attribute was the `message` field.

Enter...

## Log4J2

We are using [Log4j2](https://github.com/apache/logging-log4j2) as our logging library in our applications, configured to log everything on stdout in order to pick
up the logs later with our logging sidecar (FluentBit, you surely remember).
Now Log4j2 comes with some quirks, one of them being an internal, undocumented buffer called `log4j.encoder.byteBufferSize`. I assume this buffer exists for
some performance reason wich is probably reasonable, given logging should be as transparent to application performance
as possible. It does however throw some wrenches into the machinery of logging, because it is configured to a default
size of 8k bytes, wich causes logs that exceed this size limit to be split by the library.
This [issue on GitHub](https://github.com/OpenLiberty/open-liberty/issues/24460) covers the case quite well.

As you can imagine, this took some time (and nerves) to figure out, but once I did the fix was pretty simple.
Just throw another jvm option into the startup command of the application and everything worked like a charm:

```shell
java -jar ... -Dlog4j.encoder.byteBufferSize=262144
```

Now all logs are ingested without being split along the way. Theoretically, the story ends here, but there is one
more thing we have not covered yet.

## Just reduce the length of logs

One way of dealing with all of this could have been to reduce the length of the logs being produced in the first place.
Besides truncating logs though, there are no real options to limit the size of a stack trace for example.
The only chance in that case is filtering stack traces so that only relevant parts remain. We found that from a
stack trace, the most interesting part is the information in which of YOUR classes the exception was thrown.

So we built a log4j plugin to filter stack traces and open-sourced it too!
It is called [log4j2-filtered-stacktrace-plugin](https://github.com/Hapag-Lloyd/log4j2-filtered-stacktrace-plugin)
and available on GitHub. Feel free to check it out.

# Conclusion

So that was quite a journey, from AWS Fargate and container runtime logging drivers all the way to a plugin for Log4j2 to remove irrelevant
information from stack traces. It was quite a task to figure it all out, but it was certainly worth it.
It even gave me the possibility to contribute to OSS for the first time and write my first-ever blog post, wich I think is just great.

Cheers!
