---
layout: post
title:  "Why Continuous Profiling is a 'must have' in your Observability Strategy"
author: israel 
featured: false
categories: [ 'Cloud Native' ]
tags: [containers, devops, cloud-native, profiling, ebpf ]
image: https://user-images.githubusercontent.com/2548160/180667738-090c541e-0823-44e9-844e-e1a527f6ce6a.jpg
date:   2022-07-22 15:01:35 +0300
description: "If your cloud-native observability strategy or solution does not include continuous profiling using eBPF, then you must go back to the drawing board. Here's why..."

---


Over the last 6 months, I have been passively researching Continuous Profiling (in my private time) in the context of cloud-native observability. 

My research was underpinned by the following questions: 


1. What is Continuous Profiling and why should I care?
2. How does Continuous Profiling apply to large-scale distributed, ephemeral, compute instances? 
3. Can one truly profile "continuously" in production? Is the overhead worth the trouble?
4. Are the recent advancements in eBPF making a difference? See my post on [eBPF in Plain English](https://www.israelo.io/blog/ebpf/)
5. What does Gartner think of Continuous Profiling?
6. What are observability vendors doing about this?

I will share the answers to these questions in this blog post, albeit at a high level. 

But before I dig into the details, I would like to start with the conclusions that I drew from my research. Yes, I like Button Line Up Front (BLUF) a lot:  


## What’s the conclusion? 

Profiling is measuring what part of your application is consuming a particular resource.  **Continuous** Profiling adds time dynamics to the definition above; it provides **always-on **profiling data from app environments. This includes being able to aggregate and understand data from a production system in an observability tool in a similar way to what traditional APM tools do with instrumentation and metrics.

Continuous **distributed** profiling is the next disruptor of observability. When combined with recent advances in eBPF (particularly in the Linux user space), Continuous distributed profiling (with emphasis on **distributed**) will truly enable DevOps and SRE engineers to probe their applications (i.e. user space events) and infrastructure (hardware, network performance, security) without knowing the questions to ask in advance. It is the bedrock for “truly” discovering unknown unknowns in a distributed microservices architecture, but only a few observability vendors are doing it well. In fact, none of the big observability players has harnessed the power of eBPF in Continuous **distributed** profiling. As a matter of fact, only a few have continuous profiling features, let alone on eBPF.

## Why? 

Because most of the big observability players do profiling by instrumentation, which is synonymous with a driver who frequently takes his/her eyes off the road. Profiling by instrumentation is built on the principle of taking system snapshots based on certain pre-define events or frequency, this is analogous to a security guard on duty who only wakes up from his sleep when someone knocks at his gate. Dangerous. This is why organisations need an observability solution that continuously profiles their workloads. 

Furthermore, I have lost track of how many times I've had to set up instrumentation agents to decrease snapshot collection owing to overhead concerns.  It is always a difficult decision because the customer would like to get the profiled call graphs during a performance issue, but you cannot have the agent collect snapshots aggressively during this time because it exacerbates the performance issue. 

Instrumentation profiling is not the best. Are you familiar with Linux’s Kill -3 command? It terminates the PID, dumps the JVM core, and resumes the PID. Too much of it is not good. Anyway, that’s a different blog post topic. 

The meta-point is that organisations must shift from the dominant instrumentation profiling approach to continuous profiling solutions, and eBPF makes it possible to do this without overbearing the application and/or infrastructure. 

Furthermore, agent bytecode instrumentation does not only introduce performance overhead; it often requires app or container restarts. The multi-agent (app, infra, etc) nature of this approach also makes it cumbersome to install, configure, and upgrade agents at scale. Only Continuous Profiling on eBPF can alleviate the agent overhead concerns; it also provides continuous visibility and profiling call graphs at no extra performance overhead cost, and it eliminates the need for a separate app, infra, and sometimes security agents. eBPF provides holistic visibility into the kernel and user space. It completely eradicates the need to instrument the application, which literally means allowing someone else’s app (called agents) to attach itself to your organisation’s critical application by completely **re-transforming** your application’s bytecode. 

Did you know that agents can also go as far as re-defining your application classes? _Say what? Say what…?_ Yes, using the [Java Instrumentation API](https://docs.oracle.com/en/java/javase/11/docs/api/java.instrument/java/lang/instrument/Instrumentation.html) for example, the API contains the following commonly used methods: 


* _retransformClasses_ – facilitates the instrumentation of already loaded classes by adding byte-code
* _redefineClasses_ – redefine the supplied set of classes using the supplied class files, meaning that the class will be fully replaced, not modified as with _retransformClasses_
* _addTransformer—adds_ a transformer to the instrumentation engine
* _getAllLoadedClasses_ – returns an array of all classes currently loaded by the JVM
* _removeTransformer_ – unregisters the supplied transformer

Now you see why I fell in love with eBPF because it eliminates the need for bytecode instrumentation? Given the option, I would rather not ingest any other black-box agents into my application. 

Finally, most instrumentation profiling agents in the market today are designed to instrument only well-known libraries, function calls, and protocols such as HTTP(S), JDBC, ADO.Net, and message bus, among others. This is why I have spent hours (sometimes days) working with customers to apply **custom instrumentation** to their applications—which literally means instructing the agent to instrument a less-known protocol, background task, or some other legacy library.  

My point is, if typical app agents are pre-wired to instrument certain parts of the applications, then how can they claim to discover all unknown unknowns across the full stack? This is why I would make instrumentation an opportunity cost in favour of eBPF. eBPF is event-driven,and it allows developers to package the user space application logic to be executed in the Linux kernel space as bytecode.


## Summary 

In summary, Continuous Distributed Profiling is the future of observability because, unlike instrumentation profiling, it provides always-on visibility into the entire stack. When combined with eBPF, you can use one agent to get app visibility, infrastructure visibility, network visibility, and security visibility across the entire fleet in a safe and secure manner. 


## But there are a few drawbacks: 


* eBPF is only currently supported on Linux 4.4. Although Microsoft is actively working on adding [eBPF support to Windows](https://github.com/microsoft/ebpf-for-windows) - including eBPF support for HyperVisor-enforced Code Integrity (HVCI). 
* eBPF does not play nice with serverless technologies since customers do not have access to deploy eBPF probes to these servers. The eBPF eco-system is asking for it. There’s an [open request for AWS to support eBPF on Fargate](https://github.com/aws/containers-roadmap/issues/1027). 
* eBPF is hard to begin with. As a result, there are not a lot of Product Managers in the industry today who can fully articulate the benefits of eBPF, especially as it relates to Continuous Profiling in observability.  
* Finding Developers and Architects that have a good grasp of C, Go and a strong understanding of the inner workings of the Linux kernel can be a challenge.
* eBPF requires elevated privilege access to do its magic, but the community is generally comfortable with this because eBPF is fail-proof. It cannot break the kernel either. 

     



## Lamentation

I just realised that this is no longer a BLUF. I broke my own rule, but I didn’t want to stop my flow. Since the summary of research turned into a full-blown blog, I would have to write a part 2 post on this subject — to specifically answer the six cardinal questions above.

Watch this space! 


-------
>  **⚠ DISCLAIMER**

> Opinions expressed are solely my own and do not express the views or opinions of my employer.