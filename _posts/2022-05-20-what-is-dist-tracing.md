---
layout: post
title: "How Does Distributed Tracing Work?"
author: israel
featured: false
categories: [ 'Cloud Native' ]
tags: [containers, devops, cloud-native, tracing ]
image: https://user-images.githubusercontent.com/2548160/179272497-41d5c333-e0ce-4567-99d1-3fe654fbb152.png
date:  2022-05-20 15:01:35 +0300
description: "Distributed tracing provides end-to-end visibility and reveals service dependencies – showing how the services respond to each other to fulfil a request..."

---
<br>

## The problem 

Modern software architectures based on microservices and serverless provide benefits to application development but at the expense of decreased visibility. Teams can more easily manage, monitor, and run their specific services but quickly lose track of the whole system's behaviour. During an incident, a user may report a problem with a transaction that spans many microservices, serverless operations, and teams. It becomes practically impossible to distinguish between the service accountable for the problem and those impacted by it.

## So What is distributed tracing? 

Distributed tracing provides end-to-end visibility and reveals service dependencies – showing how the services respond to each other to fulfil a request. 

<p class="aligncenter">
<img class="lazyimg" alt="Otel configuration" src="https://user-images.githubusercontent.com/2548160/179273428-613bfcb8-d88b-4249-bda8-54a4640bc4eb.jpg"/> 
<br>
</p>

Further, distributed tracing tells the story of an end-to-end request - it provides visibility across service boundaries - what is broken and slow and which specific logs and metrics can help resolve the incident at hand. 

A single trace generally depicts the activity for a single transaction or request inside the observed application. For example, a check-out request from the browser or mobile device to the database and back. 

The ability to visualise transactions in their entirety makes it possible to compare anomalous traces to performant traces  - to identify discrepancies in behaviour, structure, and time. This information enables you to improve MTTD (Mean Time to Detection) and to better perform RCAs.  


## What is a trace? 

A trace in distributed tracing consists of a sequence of tagged time frames known as spans. A span can be thought of as a single unit of work. 

Moreso, spans have a start and end time and optionally may include other metadata like logs or tags that can help classify “what happened.” Spans have relationships between one another, including parent-child relationships, which are used to show the specific path a particular transaction takes through the numerous services or components that make up the application.

Let’s recap: 

* **Trace** represents an end-to-end request; made up of single or multiple spans
* **Span** represents work done by a single service with time intervals and associated metadata; the building blocks of a trace
* **Tags** metadata to help contextualize a span

To summarise, the primary purpose of traces is to give a request-centric perspective. So, although microservices allow teams and services to operate independently, distributed tracing offers a centralised resource that allows all teams to comprehend problems from the viewpoint of the user. 


## So how does distributed tracing work? 

To understand how distributed tracing works, consider how it handles a single request:

Tracing starts the moment an end user interacts with an application. When the user sends an initial request, such as an HTTP request, it is assigned a unique trace ID. 

As the request traverses through the fleet of microservices, every operation performed on it (called a “span” or a “child span”) is tagged with that first request’s trace ID, as well as its own unique ID, plus the ID of the operation that originally generated the current request (called the “parent span”). Think of it as a tag-and-follow game. 

Each span represents a single step in the request's journey and is decorated with metadata relating to the microservice executing that activity. These usually include:


* The service name and address of the process of handling the request.
* Logs and events that provide context about the process’s activity.
* Tags to query and filter requests by session ID, database host, HTTP method, and other identifiers.
* Detailed stack traces and error messages in the event of a failure.

When the spans are sent to a distributed tracing tool like Zipkin, Jaeger or other observability vendors, they correlate the data from all the spans and format them into visualisations that are available on request through their respective web interface. 


-------
>  **⚠ DISCLAIMER**

> Opinions expressed are solely my own and do not express the views or opinions of my employer.