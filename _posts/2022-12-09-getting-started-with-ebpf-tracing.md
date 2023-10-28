---
layout: post
title: "How to get started with CPU Profiling using eBPF"
author: israel
featured: false
categories: [ 'Cloud Native' ]
tags: [containers, devops, cloud-native, kubernetes, ebpf, cpu, profiling, flamegraph ]
image: https://user-images.githubusercontent.com/2548160/206739101-e6a6a80f-8628-4d9f-80f6-de17473e52b4.png
date:  2022-12-09 15:01:35 +0300
description: "bpftrace to flamegraph in 3 simple commmands. eBPF allows for the creation of custom programs that can be attached to various points in the kernel, allowing for detailed and efficient monitoring and analysis of system behavior."

---

<p></p>

## Why eBPF? ##

eBPF (extended Berkeley Packet Filter) is a powerful and versatile tool for observability and monitoring in Linux systems. It allows for the creation of custom programs that can be attached to various points in the kernel, allowing for detailed and efficient monitoring and analysis of system behavior.

eBPF provides a number of benefits over traditional observability tools. One of the biggest advantages is its ability to operate at the kernel level, allowing for deep visibility into the inner workings of the system. This can provide valuable insights into performance and behavior that would otherwise be difficult or impossible to obtain.

<p class="aligncenter">
[Related Article: <a href="https://www.israelo.io/blog/ebpf/" target="_blank"> eBPF in plain English </a> ]
</p>

Another key benefit of eBPF is its flexibility. eBPF programs can be attached to a wide range of kernel functions and events, allowing for a high degree of customisation and tailoring to specific needs. This makes eBPF an ideal tool for a wide range of applications, from simple performance monitoring to complex debugging and troubleshooting.

eBPF also has the advantage of being relatively lightweight and efficient. The programs are compiled into a specialized bytecode format that can be easily executed by the kernel, reducing overhead and enabling real-time monitoring and analysis. This makes eBPF a valuable tool for monitoring and optimizing systems with high performance requirements.

Despite these benefits, eBPF does have some limitations and challenges. One of the biggest challenges is the learning curve required to effectively use eBPF. The tool is powerful but can be complex, and requires a good understanding of kernel internals and system behavior. Additionally, eBPF programs can be difficult to debug and troubleshoot, especially for those new to the tool.

The objective of this blog is to give new starters a gentle hands-on introduction to eBPF tracing. These are steps that I wished I had when I got started. 

If you're new to eBPF tracing, here are some steps to get started with eBPF tracing. 

## System Setup ##

1. First, you will need a Linux OS. In my case, I used an Ubuntu <a href="https://multipass.run/" target="_blank">Multipass</a> VM on M1 MacOs. 
If you're using a multipass VM, you will need to install <a href="https://wiki.ubuntu.com/Debug%20Symbol%20Packages" target="_blank" >debug symbol packages</a> first. Alternatively, you may use this <a href="https://github.com/iovisor/bpftrace/blob/master/Vagrantfile" target="_blank">Vagrant file </a> to spin up a VM with all the bcc tools you need. 

2. Make sure your Linux kernel is up-to-date. eBPF was introduced in Linux kernel 3.15, so you'll need at least version 4.1 to use the eBPF tracing commands in this post. You can check your kernel version by running the `uname -r ` command. 


3. Install bpftrace command-line utility and the bcc collection tools. 
   
     ```cpp
     sudo apt-get install bpfcc-tools linux-headers-$(uname -r)

     ```


4. We need to visualise the trace with flamegraphs, so you'd need to clone this repo: 

```cpp

git clone https://github.com/brendangregg/FlameGraph.git

```

Let's get started! 

## CPU Profiling ##

Let's start by profiling all processes that are consuming your CPU budget.  

```cpp

sudo bpftrace -e 'profile:hz:99 { @[kstack] = count(); }' > trace.data 

```

The above command will sample all running processes 99 times per second. The on-CPU profile samples answer two fundamental questions: 

1. Where are you - This essentially interrupts the CPU cores at 99Hz and collects the CPU's answer for the "where are you" question. 
2. How did you get here - The historical information on how the CPU got to where it is super important to formulate the stack traces.

Also, the `kstack` returns the kernel stack trace. This is used as a key for the map, so that it can be frequency counted. There is also `ustack` for the user-level stack trace should you wish to use that instead. 

Let the above command run for while (min of 2 minutes is recommended), then stop it by executing `CTLR + C` . 

Let's view the stack trace 

```cpp
less trace.data

Attaching 1 probe...

@[
   {truncated}
    do_page_fault+384
    do_translation_fault+156
    do_mem_abort+76
    el0_ia+148
    el0t_64_sync_handler+292
    el0t_64_sync+420
]: 1
@[
    dict_repeat+48
    lzma2_lzma+424
    xz_dec_lzma2_run+144
    dec_block+444
    dec_main+764
    xz_dec_run+68
    squashfs_xz_uncompress+212
    squashfs_decompress+112
    squashfs_read_data+212

```

The output is a stack traces which consist of process details, and a single number - this represents the number of times this stack trace was sampled. 

### Analyse CPU stack trace using flamegraphs ### 

Next, let's visualise the stack traces in flamegrapsh so we can easily analyse the result of our CPU profile.   

A flamegraph is a visualisation tool used to profile the performance of a program. It shows a graphical representation of the call stack, with the width of each frame representing the amount of time the program spends in that particular function or method. Flamegraphs are commonly used to identify performance bottlenecks and hot spots in a program, as well as to optimise the program's performance. They can be generated using a variety of tools, including performance profilers and tracing tools.

We will use Brendan's flamegraph tool that was downloaded in step 4 above to convert the bpftrace file to a flamegraph. 

```cpp

cd FlameGraph

./stackcollapse-bpftrace.pl trace.data > trace.folded

./flamegraph.pl --inverted trace.folded > traceflamegraph.svg

```

Next, load up the traceflamegraph.svg in your browser and analyse 

<p class="aligncenter">
<img class="lazyimg" src="https://user-images.githubusercontent.com/2548160/206728975-dab2d420-6cbe-4039-934e-8e18239410f1.svg"/> 
<br>
</p>


## PPROF ##

Most profiling tools uses the <a href="https://github.com/google/pprof" target="_blank">pprof</a> format it is a convenient and efficient way to store and share profiling data, and is widely used in the world of software engineering.

There are several other reasons to use the pprof format for profiling:

1. pprof is a standardised format that is supported by many different profiling tools, so you can use the same format regardless of which tool you are using. This makes it easier to share and compare profiles generated by different tools.

2. pprof is designed to be a compact, efficient format for storing profiling data. This makes it faster and easier to transfer and store large profiling datasets.

3. pprof includes support for a variety of different types of profiling data, including CPU, memory, and network usage. This makes it a versatile format that can be used for a wide range of profiling tasks.

4. pprof is a text-based format, which makes it easy to read and manipulate using simple command-line tools. This can be useful for quickly extracting and analyzing specific pieces of information from a profile.

So instead of the folded file format example above, you may consider converting your profiling data into pprof format instead.


## Summary ##

In summary, eBPF tracing can be a powerful and versatile tool for Linux users. With a little bit of setup and the right eBPF programs, you can monitor and analyze your system in ways that were previously impossible. 


Give it a try and see what insights you can uncover!


## Refs ##

1. https://github.com/iovisor/bpftrace 
2. https://www.brendangregg.com/flamegraphs.html
3. https://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html 
4. https://wiki.ubuntu.com/Debug%20Symbol%20Packages  
5. https://github.com/iovisor/bpftrace/blob/master/docs/tutorial_one_liners.md#lesson-9-profile-on-cpu-kernel-stacks
 

-------
>  **⚠ DISCLAIMER**
> Opinions expressed are solely my own and do not express the views or opinions of my employer.


