---
title: "Thread pool problems on OpenLiberty"
description: 'This blog article describes how thread pool issues can happen in the automatically managed default executor of OpenLiberty'
pubDate: 'Aug 20 2025'
image: '/openliberty_threading.jpg'
tags:
  - Java
  - OpenLiberty
  - WebSphere
  - Cloud
  - threading
---

I have been working on an application which is part of one of our most requested systems and workflows.
This application is written in Java 11 and uses OpenLiberty with Jakarta 10 and all the specifications below it.
It is deployed as a dockerized container in a Kubernetes environment. In theory, it is an absolutely boring application,
which is only supposed to read from its database on request time and then call another downstream service via HTTP.

That same code has been around forever as part of another application on a mainframe, and the performance there was
about as boring as the code itself. Zero latency spikes, p50 and p90 about the same (<100 ms difference) and it worked
fine since years. But since it has been running on a mainframe and not on the cloud, it had to move.

## Problems from the get-go

Let me start by saying that I am a big proponent of the cloud. It's an amazing tool for organizations looking to modernize
their organizational structure and processes, especially in fast-changing and hard-to-predict environments where infrastructure
requirements change all the time.

Applications in the cloud behave a little different, however, and one must be aware of that to successfully
take advantage of it.

In our case, our application was previously running in an environment where resources were practically unlimited.
This changed now with the code running on Kubernetes pods, each of which had one vCPU and 2 GB of memory.

What we observed when comparing the mainframe version vs. the cloud version was a dramatically increased p90 latency (about 10x worse)
and a slightly worse p50 performance. Infrastructure metrics did not show any sign of CPU or memory limitations with
roughly 5% to 10% CPU usage during peak hours and ~25% memory usage, which is absolutely not alarming.

## Digging deeper

Since compute must not have been the bottleneck in this scenario, I looked elsewhere. Good candidates for application
slowdowns are usually any sort of resource pools. Connection pools or thread pools directly come to mind.

In our case the connection pool was already increased and saw around 10% of active connections with about 90% to spare.
The thread pool usage, however, looked strange: The size of the thread pool directly matched the number of active threads
in any given minute. This was a pattern I have not seen previously and led me to investigate OpenLiberty's threading
concept.

OpenLiberty has a so-called __self-tuning algorithm__ that constantly adjusts the size of the default thread pool.
They say that...

_you do not need to manually tune the thread pool for most applications, in rare cases you might need to configure thread pool settings_
([thread pool tuning](https://openliberty.io/docs/latest/thread-pool-tuning.html))

It is an interesting concept! Every 1.5 seconds, metrics are evaluated to determine whether the current thread
pool size still fits the application requirements. This should, in theory, lead to optimal resource usage with very
little performance impact.
The default thread pool size is set to `min(2 * # vCPUs, 4)`, which in our case equates to 4, as we only have 1 vCPU.

## Observations

In practice, this automagic tuning of the thread pool size led to problems. One request against our application
took a lot less than 1.5 seconds (usually around 300 ms). Additionally, the http calls to the downstream service
are implemented asynchronously, meaning they run in their own thread. One request causes three downstream requests.
You can see where this is going: As soon as two requests hit the same pod during one second, the thread pool is immediately
exhausted, as all four threads are already blocked by the first request. This causes the second request to wait until
either the thread pool grows (due to magic) or the first request is completed, whichever is first.

A pretty nasty situation to be in!

## Actions

To validate my hypothesis, we have temporarily increased the number of pods to such a high number, that each
individual instance was handling less than one request per second. This caused the latencies to immediately drop to
levels which were identical to the mainframe version of the application.

In addition, we have tested the same resource configuration with larger instances (but less of them) and this yielded
the same result.

Finally, we have increased the minimum thread pool size of the default executor by adding the following line to our
`server.xml` file:

```xml
<executor coreThreads="48" />
```

This also resolved the latency issues completely, even with less resources available in total.

Adding the same line to other applications within the company has also resulted in a performance boost regarding p90
latencies, so this issue was affecting all our applications using OpenLiberty!

## Takeaways

1. The self-tuning algorithm for the thread pool size of the OpenLiberty server is a great starting point for most
   applications. However, if you are doing I/O or blocking things on the default executor thread pool, beware that
   you will quickly run into thread pool limits, especially on smaller machines.
2. Look for resource limits other than CPU or Memory. Even if metrics suggest no infrastructure problem,
   the application itself might manage resource pools which can be exhausted too.
3. Make sure to design for the cloud. Any assumption regarding performance must be revalidated, especially in
   enterprise environments where the organization and the knowledge might still be tuned to a mainframe environment.
