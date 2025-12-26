---
layout: post
title: Observability is not monitoring, and definitely not dashboards
img: monitoring-observability.png
tags: [observability]
---

From my experience, there is a lot of confusion between monitoring and observability.  
Most of the time, the two terms are used wrongly or even worse, used interchangeably.  
You will hear team calling Grafana + Prometheus, their “observability” stack but let’s be honest, this a `monitoring` stack.  

Most teams think they have `observability` but most of them don’t.  
They have dashboards.  
They have alerts.  

## Monitoring vs Observability

First let’s clarify those two terms.

### Monitoring

Monitoring is about tracking metrics for known failures, like CPU spikes, p95 latency, error rates, etc…  
It’s all about asking questions you already know to ask.  
Ex: “Is the latency high?”, “Is the CPU skyrocketing?”  
All predefined dashboards, alerts aggregated metrics are considered as `monitoring`  
It works great for known issues but what about the unknowns ones?  

### Observability

In the opposite, observability is all about the unknowns, the questions you didn’t know you needed to ask.  
It gives you the ability to understand the internal state of the system by analyzing the data it generates,  
such as logs, metrics and traces, without any modification.  
Ex: “Why are retries increasing?”, “Which dependency is the bottleneck right now?”  

Metrics and dashboards alone cannot answer these questions.  
They show symptoms but not explanations.  
That's what `observability` is about: Understand the unknowns!  

tl;dr
> Monitoring answers known questions, observability answers the unknowns ones

## The dashboards trap

You have a Grafana dashboard for every service, every team has dashboards, maybe even a dashboard of dashboards.  
Teams often feel confident because all those dashboards exist and even claim they have “full observability”.  
Unfortunately, that confidence is only an illusion,  `observability` has been confused with `monitoring`.  
Don’t get me wrong, dashboards are great and useful but dashboards alone will fall short during incidents.  

## Telemetry is not observability

Another misconception, collecting metrics, traces and logs is not observability.  
They are indeed the 3 pillars of observability but don't magically give you observability, they are just raw data.  
Observability is not about collecting data but how they are structured:

- High-cardinality
- Rich context
- Correlation between data
- Structured logging
- Slice and pivot in real time
For example during an incident, you should be able to break down latency by user, request type, version, feature flags, etc...  
In real time you can answer any question to investigate on the incident, that's real observability!  

## Why you need real observability?

As we saw before `monitoring` helps with the known questions but most of the time system fails in a way you didn't predict.  
It's 3 AM, you got paged, a major incident is happening: you don't have time to build new dashboards or wait for aggregated metrics to update.  
You need to ask arbitrary questions about your current system's behavior.  
You want to answer the unknowns and that's where `observability` helps you.  
With real observability, you can slice your data in any dimension, you can trace requests across all your microservices,  
you can filter but also correlate logs, etc...

It's not one or the other, you need both `monitoring` and `observability` to guarantee **availability**, **reliability** and **durability**.  

### Cost

Yes, high-cardinality has a cost, and not a small one but downtime costs more.  
High-cardinality doesn't mean store everything "just in case", you need to be selective on what matters for your system.  
The most common would be:

- Request ID / Trace ID
- Request type
- Feature flags
- Version
- User or other ID

You don't need to store the full request and/or response body for example.  
Observability is not about having more data, it's about reducing time to understand failure.  

With observability, mindset need to change, we are not asking "is server X healthy?" or "is pod Y healthy?".  
No, what you want to know is: "Why did request X from user Y failed?", "Why latency is slow in region Z?"  
That mindset shift needs to be done by the full team, not only a few, otherwise your high-cardinality will be inconsistent.  

Like for quality, observability is not a one time thing, you gather what you think is important and you keep re-iterating on it.  

## Litmus test

If you can confidently answer yes to those criteria, you're on the path of `observability`  

- You don't need to deploy new tool(s) or code to understand an incident that is happening
- You are able to understand the incident in a timely manner
- You can verify your hypotheses right away
- You don't have 10 different tools to look at while trying to correlate things

## Conclusion

I will say it again, both `monitoring` and `observability` matters.  
Both have a different job:

- Monitoring tells you something is off
- Observability tells you why and how to fix it

So, do you have real observability?  

I hope this article allowed to clarify the difference between both.
