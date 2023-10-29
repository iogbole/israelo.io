---
layout: post
title:  "Observability vs Monitoring: A DevOps perspective"
author: israel
featured: false
categories: [ 'Cloud Native' ]
tags: [containers, devops, cloud-native, kubernetes ]
image: https://user-images.githubusercontent.com/2548160/147579433-1c8ead27-8c3a-436c-adc0-3ac194fc7bfe.jpg
date:   2021-12-28 15:01:35 +0300
description: "Observability does not replace monitoring. They are NOT mutually exclusive; rather, they are mutually reinforcing. Although observability and monitoring are not the same, they are often used interchangeably by vendors and customers alike."
---

## Bottom Line 
- Monitoring is based on gathering predefined sets of metrics or logs. It helps organisations to know what is broken and why it is broken. 
- Monitoring extends observability. 
- Observability is anchored on the discovery of unknown traits and patterns. It uncovers unknown-unknowns. 
- Observability uses the output of a system (i.e. MELT data) to determine the internal state of the observed system. It is not a new concept. 
- Observability does not replace monitoring. They are NOT mutually exclusive; rather, they are mutually reinforcing.
- Observability precedes monitoring. That is, you start monitoring after a system is observed. 
- Observability is best suited for providing insights into complex, distributed microservices architecture due to the non-linearity and unpredictability of failures in modern applications. 

-------
When organisations adopt a DevOps culture, they typically start by decomposing monolithic applications into microservices architectures to improve scalability, deployment cadence, fault isolation, etc. Traditional Application Performance Monitoring (APM) tools are typically designed to address known metrics (e.g Avg. response time) in predefined entities, and as a result, they are incapable of providing "true" end-to-end visibility in modern distributed applications.

As cloud-native microservices architectures become the norm for most modern applications, effective tracing and debugging require that the application is <b>observable</b>. Consequently, the need for observability is becoming increasingly important as businesses strive to scale their DevOps strategies to keep up with the ever-increasing complexity of the software delivery process.

Observability is crucial in any DevOps-oriented organisation, it is, however, often confused with <b>monitoring</b>. Both terms are often used interchangeably by vendors and users alike, despite the fact that they are not the same. Also, Observability and monitoring are not mutually exclusive. 

The purpose of this post is to demystify the differences and relationships between monitoring and observability.

## What is Monitoring? 
A monitoring tool must answer two fundamental questions - "what is broken, and why?" (source: <a href="https://sre.google/sre-book/" target="_blank"> Google SRE book </a>). A monitoring tool provides critical insights and information about an application's performance and usage trends - this includes information on memory issues, code bottlenecks, availability, server health, end-user experience, and much more. 
Refer to " <a href="https://sre.google/sre-book/monitoring-distributed-systems/" target="_blank"> Chapter 6 - Monitoring Distributed Systems </a> of the <b> Google SRE book </b> for details. 

Furthermore, monitoring provides insights into how an application, network, or infrastructure is performing. It is critical for building operations and business dashboards, creating health rules (using static thresholds or dynamic baselines), analysing usage and system performance trends, etc.

Monitoring, on its own, has a disadvantage in a complex microservices architecture since production failures are non-linear and difficult to predict owing to the distributed nature of microservices. The ephemeral nature of microservices further complicates issues for traditional monitoring techniques. Despite this disadvantage, monitoring remains essential for operations, DevOps and SRE. If the monitored metrics and health rules are simple and focused on actionable data, they will provide the business with a good picture of how healthy the system is - by answering the "what is broken, and why" question.

To summarise, monitoring helps organisations to: 
- Detect system issues: It alerts users to problems or displays them on dashboards. It improves Mean Time to Detection (MTTD)
- Problem Resolution: It improves Mean Time to Repair (MTTR) and aids Root Cause analysis (RCA) of problems. 
- Continuous Improvement: It enhances capacity planning, financial planning, trend analysis, etc.

## What is Observability? 
While monitoring answers the "what is broken and why?" question, observability uncovers the unknown-unknows. Observability orgininated from a <b>control theory </b> (source <a href="https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-241j-dynamic-systems-and-control-spring-2011/readings/MIT6_241JS11_chap24.pdf" target="_blank">  MIT : Dynamic Systems and
Control </a> ). 


> The objective of the Control Theory is to develop a model or algorithm governing the application of system inputs to drive the system to the desired state, while minimising any delay, overshoot, or steady-state error and ensuring a level of control stability; often with the aim to achieve a degree of optimality.

<i> Source <a href="https://en.wikipedia.org/wiki/Control_theory" target="_blank"> Wikipedia: Control Theory </a> </i>

Observability, therefore, is a measure of how well a system's internal state can be deduced from its external characteristics or outputs. In this context, the `internal state` refers to the unknown-unknows, or unpredictability/non-linearity of a failure in a distributed system, and the `outputs` refers to the Metrics, Events, Logs and Traces (aka MELT) data from the observable system. Thus, <b> MELT data forms the pillars of observability </b>

<p class="aligncenter">
<img class="lazyimg" src="https://user-images.githubusercontent.com/2548160/147602582-abbee2bb-f030-4f3f-95cd-23b9b5329b1e.jpg"/> 
<br>
</p>
<i>Source <a href="https://peter.bourgon.org/blog/2017/02/21/metrics-tracing-and-logging.html"> metrics-tracing-and-logging</a> </i>
 
Unlike monitoring, observability is a measure of a system's ability to diagnose what is going on inside rather than a tool used to measure the system's performance. In other words, when a system is observable, it allows for the measurement and inference of its internal state; this provides an additional context to allow you to get to the underlying cause of problems quicker. The additional/enriched context provided by observability makes it more suitable for acquiring insights into the internals of microservices architecture.

To summarise, observability answer the following questions: 
- Which services did a request traverse, and where did performance bottlenecks occur?
- How did the request's execution deviate from the intended system behaviour?
- Why did the request fail?
- How was the request processed by the service?

## How are Observability and Monitoring different?

Observability and monitoring have symbiotic relationship. One may argue that monitoring extends observability. This section explores common differences between observability and monitoring. 
 
#### Mutual Exclusivity 
Observability and monitoring are NOT mutually exclusive. Observability precedes monitoring; that is, you set up your monitoring after a system has been observed. In other words, observability is a superset of monitoring. A system can be observed in many ways. However, the most popular method for observing applications is via instrumentation, which injects agents into the application's byte code. The agent can either be vendor-specific or an opensource agent - such as the <a href="https://opentelemetry.io/docs/collector/getting-started/" target="_blank"> OpenTelemetry </a> agents. 

#### Purpose  
Observability and monitoring are complementary, with each fulfilling a different purpose. 

Monitoring uses a sampling mechanism to collect data such as response time, requests, downtime, bottlenecks, and so on to track the overall health of a system. On the other hand, observability provides additional contextual information into a failure by tracing the requests through different microservices. 

#### Addressing the unknown-unknowns
Monitoring seeks to report known and predictable failures, while observability seeks to detect issues that users have not yet found. The underpinning principle of observability is to handle known failures and identify incoming issues before the users discover them.

## Conclusion
Although observability and monitoring are not the same, they are often used interchangeably by vendors and customers alike, which creates lots of confusion.
Monitoring and observability have distinct objectives. Observability does not replace monitoring. They are NOT mutually exclusive; rather, they are mutually reinforcing.

Monitoring tracks the overall health of a system, and it is best suited to measure known and limited KPIs and failure modes. Observability, on the other hand, aims to provide highly granular insights into the behaviour of a system. The rich context that observability provides make it more suitable for cloud-native microservices architecture.


-------
> **⚠ DISCLAIMER**  
> Opinions expressed are solely my own and do not express the views or opinions of my employer.
