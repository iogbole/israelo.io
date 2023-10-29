---
layout: post
title:  "Getting Started with eBPF: Monitoring TCP Retransmissions Using eBPF, Go and Prometheus"
author: israel
categories: [ 'Cloud Native' ]
featured: true
tags: [containers, devops, cloud-native, kubernetes, ebpf ]
image: 'https://user-images.githubusercontent.com/2548160/274512682-f06a933c-16e3-4288-a1fa-80237cc4b16d.png'
date:   2023-10-12 09:01:35 +0300
description: "This is a beginner's guide to eBPF. Learn how to use eBPF, Go, and Prometheus to monitor network events unobtrusively. You'll learn how to use Linux tracepoints to monitor TCP transmissions and use Prometheus to store and visualize the data you collect."

tags:   [ebpf, observability]
---
As a Product Manager in tech, I strongly believe it is essential to understand the technology that underpins the products I manage. This knowledge not only allows me to keep pace with the evolving industry trends but also enriches my interactions with both customers and my engineering counterparts.

Recently, I've been working on a product that uses Extended Berkeley Packet Filter (eBPF). eBPF is a revolutionary technology that allows users to extend the functionality of the Linux kernel without having to modify the kernel code itself. Intrigued to learn more, I got a copy of Liz Rice's book, ["Learning eBPF"](https://isovalent.com/books/learning-ebpf/). The book is so enlightening that I couldn't resist rolling up my sleeves to get hands-on with this revolutionary technology, albeit a little.

Further, a specific focus for me has been the use of eBPF for monitoring TCP retransmissions, which can occur when a TCP segment goes unacknowledged by its receiver within a designated time frame. My interest in TCP retransmissions stems from a challenging experience troubleshooting intermittent connectivity issues with an APM agent in a customer's production environment, in a previous role. Had eBPF been in my toolkit back then, that painful issue would have been far easier to diagnose and resolve.

This blog aims to chronicle my exploration of eBPF and Go and is targeted at anyone interested in learning eBPF. We will delve into the fundamentals of monitoring network events using eBPF, Go, and Prometheus. 

> The source code is available at [https://github.com/iogbole/ebpf-network-viz](https://github.com/iogbole/ebpf-network-viz)


Let's begin by defining the problem. 

## The Ghost in the Network: TCP Retransmissions

TCP retransmissions aren't inherently bad; they're a fundamental part of how TCP/IP networks function. However, when they occur frequently, they can signify network issues that lead to poor application performance. A high number of retransmissions can cause:

* **Increased Latency**: Packets have to be sent again, which takes extra time.
* **Higher CPU Usage**: Both sending and receiving systems have to do additional work to handle the retransmissions.
* **Bandwidth Inefficiency**: Retransmissions consume bandwidth that could be better used by new data.
* **User Experience Degradation**: All the above contribute to a laggy or suboptimal user experience.

<p align="center">
<img width="600" alt="tcp retransmission" src="https://github-production-user-asset-6210df.s3.amazonaws.com/2548160/273732239-ec8dd025-ea85-4e7f-9ef3-0063ff75f1e0.png">
</p>

Imagine working on a high-speed, low-latency product and encountering intermittent slowdowns in data transmission. This situation can be tricky to diagnose and could bring your product to its knees. When I faced this issue, I took it upon myself to delve deep and understand what was happening under the hood. **Wireshark led me to the root cause: excessive TCP retransmissions** due to a faulty firewall policy.

One can easily trigger TCP retransmission, by executing: 

```c
sudo tc qdisc add dev eth0 root netem loss 10% delay 100ms
```
and it will surely mess up your network performance and introduce high CPU usage. I was once crazy enough to use 50% on an EC2 instance and it booted me out of SSH connection until I restarted the node via the console.  **Do not try this out at home ;)** 

The goal of this experiment is simple: to collect all TCP retransmissions from the kernel and push the metrics to Prometheus, so that you can slice and dice the data as well as generate alerts if the retransmission rate exceeds a threshold. 

<img width="1510" alt="prom" src="https://user-images.githubusercontent.com/2548160/274725653-9b2ac550-01cc-4015-befb-9539a9b38d03.gif">


## Why eBPF? 
eBPF is a revolutionary technology that allows users to extend the functionality of the Linux kernel without having to modify the kernel code itself. It is essentially a lightweight, sandboxed virtual machine that resides within the kernel, offering secure and verified access to kernel memory.

Moreso, eBPF code is typically written in a restricted subset of the `C` language and compiled into eBPF bytecode using a compiler like Clang/LLVM. This bytecode undergoes rigorous verification to ensure that it cannot intentionally or inadvertently jeopardize the integrity of the kernel. Additionally, eBPF programs are guaranteed to execute within a finite number of instructions, making them suitable for performance-sensitive use cases like observability and network security.

Here are some of the key benefits of using eBPF:

* **Safety and security**: eBPF programs are sandboxed and verified, which means that they cannot harm the kernel or the system as a whole.
* **Performance**: eBPF programs are extremely efficient and can be used to implement complex functionality without impacting system performance.
* **Flexibility**: eBPF can be used to implement a wide range of functionality, including network monitoring, asset discovery, security, profiling, performance tracing, and more.

Functionally, eBPF allows you to run this restricted C code in response to various events, such as timers, network events, or function calls within both the kernel and user space. These event hooks are often referred to as 'probes'—`kprobes` for kernel function calls, `uprobes` for user-space function calls, and `tracepoints` for pre-defined hooks in the Linux kernel.

In the context of this blog post, we'll be focusing on `tracepoints`, specifically leveraging the <code><em>tcp_retransmit_skb</em></code>  tracepoint for monitoring TCP retransmissions.  

If you are completely new to eBPF, I recommend checking out the resources in the reference section below, starting with [What is eBPF](https://ebpf.io/what-is-ebpf/)?

## Preparation and Environment Setup
Before we begin, it's important to have your development environment properly configured. While this blog isn't an exhaustive tutorial, I'll outline the key prerequisites briefly. 

### **Using Lima on MacOS**
If you're a MacOS user like me, Lima is an excellent and easy way to emulate a Linux VM. To kick things off with Lima, follow these steps:

1. [Install Lima](https://lima-vm.io/docs/installation/) and launch it with the [ebpf-vm.yaml](https://github.com/iogbole/ebpf-network-viz/blob/main/ebpf-vm.yaml) file:

    ```cpp
    limactl start ebpf-vm.yaml
    limactl shell ebpf-vm
    ```
2. If you use Visual Studio Code, you can connect to the Lima VM via SSH:

    ```cpp
    limactl show-ssh ebpf-vm
    ```
    Subsequently, use the SSH command to link up with the remote server from the VS Code on your host machine. Lima handles file sharing and 
    port forwaring automatically. 

3. After establishing the connection, clone the required repository:

    ```cpp
    git clone https://github.com/iogbole/ebpf-network-viz.git
    ```

### **Manual Setup on Linux**

If you’re opting for a manual setup on Linux, refer to the script section in the [ebpf-vm.yaml](https://github.com/iogbole/ebpf-network-viz/blob/main/ebpf-vm.yaml#L18) file.

With your environment now primed, you’re all set to delve into the fascinating world of eBPF!

## The Solution

The diagram below depicts the solution. 

<p align="center">
<img width="1510" alt="the solution" src="https://user-images.githubusercontent.com/2548160/274510771-99bb4583-c7be-4e3e-83fc-283ea99d0195.png">
</p>

### Overview of the Components

This is how the code works at a very high level:

1. **Bytecode Loaded by Go**: The eBPF bytecode is loaded into the kernel using a Go program, which makes use of the `github.com/cilium/ebpf` package.

2. **eBPF Code Hooks to Tracepoints**: The eBPF program uses the `tracepoint/tcp/tcp_retransmit_skb` to monitor TCP retransmissions. This allows the code to trigger whenever a TCP packet is retransmitted.

2. **Collect Retransmission Events**: The data relating to the retransmitted packets—such as IP addresses, ports, and the protocol family are collected in a structured manner.

4. **Use of eBPF Maps**: eBPF maps are used to communicate between the eBPF code running in the kernel and the Go application running in user space.

5. **Perf Buffer**: A perf event buffer is used to read the events generated by the eBPF code.

6. **Exposed to HTTP**: The Go application exposes the metrics over HTTP on port 2112.

7. **Prometheus Scrapes Metrics**: Finally, Prometheus is configured to scrape these exposed metrics for monitoring or alerting purposes.


### Anatomy of the eBPF C Code
source: [retrans.c](https://github.com/iogbole/ebpf-network-viz/blob/main/src/ebpf/retrans.c)

#### BPF CO-RE 

The headers are essential for the program to function correctly. Notably, [`vmlinux.h`](https://github.com/iogbole/ebpf-network-viz/blob/main/src/ebpf/vmlinux.h) is a header generated by BPF CO-RE. BPF CO-RE (Compile Once, Run Everywhere) enhances the portability of eBPF programs across different kernel versions. It resolves as much as possible at compile time, using placeholders for kernel-specific information that can only be determined at runtime. When the program is loaded into the kernel, these placeholders are populated with actual values. This flexibility eliminates the need for recompilation when deploying on different kernel versions. Through BPF CO-RE, the `vmlinux.h` header is generated to represent kernel structures, making it easier to write eBPF programs that are not tightly bound to specific kernels.

```c
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>
#include <bpf/bpf_tracing.h>
```

To generate the `vmlinux.h` file, execute: 


```cpp
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```

#### Data Structures

The `event` and `tcp_retransmit_skb_ctx` structures are defined to hold the information related to TCP retransmissions. The structures collect various fields such as timestamps, process IDs, source and destination ports, and more.

```c
struct event {
    __u64 timestamp;
    __u32 pid;
    __u16 sport, dport;
    __u8 saddr[4], daddr[4];
    __u8 saddr_v6[16], daddr_v6[16];
    __u16 family;
    int state;
};

// Struct to receive context from tracepoint
struct tcp_retransmit_skb_ctx {
    __u64 _pad0;
    void *skbaddr;
    void *skaddr;
    int state;
    __u16 sport;
    __u16 dport;
    __u16 family;
    __u8 saddr[4];
    __u8 daddr[4];
    __u8 saddr_v6[16];
    __u8 daddr_v6[16];
};
```

##### **Finding Data Structures for Other Tracepoints**

Understanding the data structures associated with tracepoints is a key aspect when you're diving into eBPF programs for monitoring or debugging. While I focused on the `tcp_retransmit_skb` tracepoint in this blog, you may wish to explore other tracepoints. Here's how you can discover the necessary data structures for those:

1. **Locate Tracepoint Definitions**: Typically, tracepoints are defined within the Linux Kernel source code. The definitions can usually be found under `/sys/kernel/debug/tracing/events/` directory on a Linux system with the tracing subsystem enabled. Navigate through the folders to find the tracepoint of interest.
   
2. **Reading Format Files**: Within each tracepoint directory, you'll find a `format` file that describes the event structure. This will provide you with the types and names of the fields that are available for that particular tracepoint.

    ```cpp
    cat /sys/kernel/debug/tracing/events/tcp/tcp_retransmit_skb/format
    ```

    This will display the format for the `tcp_retransmit_skb` tracepoint as an example.

By familiarising yourself with the format files and possibly the kernel source code, you can create or adapt eBPF programs to tap into a wide range of system events, not just TCP retransmissions.


#### BPF Maps

The BPF map `events` is defined as a perf event array. This map serves as a communication channel between user space (Go program) and kernel space (eBPF program).

```c
struct {
    __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
    __uint(key_size, sizeof(__u32));
    __uint(value_size, sizeof(__u32));
} events SEC(".maps");

```

#### Tracepoint Function

The function `tracepoint__tcp__tcp_retransmit_skb` is attached to the `tcp_retransmit_skb` tracepoint. Here, various fields are read and stored in an `event` structure.

```c
SEC("tracepoint/tcp/tcp_retransmit_skb")
int tracepoint__tcp__tcp_retransmit_skb(struct tcp_retransmit_skb_ctx *ctx) {
 // ... code logic
}
```

#### Compiling the eBPF Code

Compile the eBPF program using the script [`run_clang.sh`](https://github.com/iogbole/ebpf-network-viz/blob/main/run_clang.sh):

```cpp
clang -O2 -g -target bpf -c ./ebpf/retrans.c -o ./ebpf/retrans.o -I/usr/include -I/usr/src/linux-headers-$(uname -r)/include  -D __BPF_TRACING__

```

### The Go Program

source: [main.go](https://github.com/iogbole/ebpf-network-viz/blob/main/src/main.go)

#### Import Packages

The code starts by importing necessary Go packages including eBPF and Prometheus libraries.

```go
import (
    "github.com/cilium/ebpf"
    "github.com/prometheus/client_golang/prometheus"
    // ... other imports
)
```

#### Loading the eBPF Program

Here, the eBPF bytecode is loaded from the `.o` object file. I opted to load the eBPF bytecode from a pre-compiled .o object file. This object file contains the bytecode of our eBPF program, which is what gets executed within the kernel. I chose this approach to maintain a clear separation of concerns: the compilation of the eBPF program is distinct from its execution. Other examples I have seen use gobpf libraries to load the C code at compile time - this approach might be easier from a CI/CD build process. 

```go
func main() {
	// Load eBPF program
	spec, err := ebpf.LoadCollectionSpec(objFileName)
	if err != nil {
		panic(err)
	}

	coll, err := ebpf.NewCollectionWithOptions(spec, ebpf.CollectionOptions{
		Programs: ebpf.ProgramOptions{
			//Verbose to catch eBPF verifier issues
			LogLevel: 1,
			LogSize:  65535,
		},
	})
  ....
```

#### Attach to Tracepoint

The program attaches to the `tcp_retransmit_skb` tracepoint using the `link.Tracepoint` function.

```go
tp, err := link.Tracepoint("tcp", "tcp_retransmit_skb", prog, nil)
```

#### Perf Event Buffer

A perf event buffer is set up to read events from the kernel space.

```go
// Set up the perf buffer to receive events
	events, err := perf.NewReader(coll.Maps["events"], os.Getpagesize())
	if err != nil {
		panic(err)
	}
	defer events.Close()
```

The Perf Event Buffer plays an essential part in bridging the gap between user-space and kernel-space communication. This buffer is a data structure that's set up to read events directly from the kernel. Essentially, it acts as a queuing mechanism, holding data that your eBPF program collects from various probes until your user-space application is ready to process it.

Here's how it generally works: 

1. Your eBPF program attaches to specific kernel functions or tracepoints and collects data, such as packet information in the case of networking or syscall information for system-level observability.

2. This data is then pushed to the Perf Event Buffer.

3. Your user-space application, written in Go, in this case, then reads from this buffer to retrieve the data for further analysis or action.

The **[BPF ring buffer](https://www.kernel.org/doc/html/next/bpf/ringbuf.html)** is a more modern alternative to Perf Event buffers, suitable for newer Kernel version

### Exposing Metrics to Prometheus 

To expose the metrics gathered by your eBPF program for monitoring, I decided to use Prometheus in the setup. Here's what I did: 

#### Prometheus Metrics Definition

Firstly, define the events and metrics that Prometheus will scrape. In this instance: 

```go
var tcpRetransmissions = promauto.NewCounterVec(prometheus.CounterOpts{
    Name: "tcp_retransmissions_total",
    Help: "Total number of TCP retransmissions",
}, []string{"ip_version", "src_ip", "src_port", "dst_ip", "dst_port"})

```

#### Starting the HTTP Server

After defining the metrics, the next step is to expose them through an HTTP endpoint. This is done by starting an HTTP server and mapping the `/metrics` path to a Prometheus handler:

```go
// Start HTTP server for Prometheus scraping
http.Handle("/metrics", promhttp.Handler())
go func() {
    if err := http.ListenAndServe(":2112", nil); err != nil {
        panic(err)
    }
}()

```

In this example, the HTTP server listens on port 2112, and Prometheus is configured to scrape metrics from this endpoint. When Prometheus accesses the `/metrics` path, it invokes the `promhttp.Handler()`, which in turn retrieves the metric data stored in `tcpRetransmissions`. This makes the data available for Prometheus to collect. 

By combining these two components, you create a seamless pipeline that collects, exposes, and monitors TCP retransmission metrics in near real-time.

#### The Event Loop and Metrics Update

The heart of the go code lies in the event loop, which continuously polls for new events from the perf event buffer. Each incoming event is processed and the relevant Prometheus metrics are updated accordingly.

The loop employs the `events.Read()` method on the perf buffer to listen for new incoming events:

```go
// Listen for events from the perf ring buffer
fmt.Println("Monitoring TCP retransmissions...")
for {
    select {
    case <-sig:
        fmt.Println("\nReceived signal, stopping...")
        return
    default:
        record, err := events.Read()
        if err != nil {
            if perf.IsUnknownEvent(err) {
                continue
            }
            panic(err)
        }

        event := tcpRetransmitEvent{}
        err = binary.Read(bytes.NewReader(record.RawSample), binary.LittleEndian, &event)
        if err != nil {
            panic(err)
        }
...
```

Upon receiving an event, the loop processes it and updates the Prometheus `tcpRetransmissions` metric. The specifics of this processing depend on the structure and content of the events, which are designed to capture various data fields such as timestamps, process IDs, source and destination ports, and so forth.

To summarise, the event loop, in combination with the previously described Prometheus setup, allows the system to efficiently collect, process, and expose metrics for TCP retransmissions in a manner that is  consistent with other monitoring and observability tools.

Next, ensure the go code works: 

```cpp 
sudo go run ./src/main.go
```

This is also a good time to confirm that the Go HTTP server is up and running: 
<p align="center">
<img width="1510" alt="go http server" src="https://user-images.githubusercontent.com/2548160/273732043-9f3ba1d3-1059-4bba-8ac5-715ecf73e817.png">
</p>


### Setting Up Prometheus in the Lima VM using nerdctl

Since the development environment is within a Lima VM, it's advantageous to leverage `nerdctl` for container management. `nerdctl` is a Docker-compatible CLI tool for containers, which is already bundled with Lima. Here's how to set up Prometheus using a custom configuration and a shell script for automation.

#### Prometheus Configuration: `prometheus.yml`

source: [prometheus.yml](https://github.com/iogbole/ebpf-network-viz/blob/main/prom_config/prometheus.yml)

The `prometheus.yml` configuration specifies how often Prometheus scrapes metrics and from where. In this case, it is configured to scrape the metrics exposed by the Go application running on port 2112. 

Here's the content of `prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'TCPRetrans'
    static_configs:
    - targets: ['127.0.0.1:2112']
```

#### Shell Script for Automated Setup

The shell script performs several tasks to ensure Prometheus runs correctly:

1. **Getting the IP Address**: The script first retrieves the IP address of `eth0` on the host machine.

2. **Updating Configuration**: It then replaces the IP address in the `prometheus.yml` configuration file to point to the correct address where the Go application is exposing metrics.

3. **Running Prometheus**: Finally, it runs the Prometheus container using nerdctl, mapping it to port 9090.

```cpp
#!/bin/bash

IP_ADDRESS=$(ip -4 addr show eth0 | grep -oP '(?&lt;=inet\s)\d+(\.\d+){3}')

# Replace the IP address in the prometheus.yml file.
CONFIG_FILE="prom_config/prometheus.yml"
sed -i "s/[0-9]\+\.[0-9]\+\.[0-9]\+\.[0-9]\+:2112/${IP_ADDRESS}:2112/g" "$CONFIG_FILE"
echo "Updated prometheus.yml with IP address: $IP_ADDRESS"
sleep 3
nerdctl run --rm -p 9090:9090 -v "$PWD/prom_config:/etc/prometheus" prom/prometheus
```

This script automates the process, making it easier to deploy Prometheus within your Lima VM. 

Since Lima also does automatic port forwarding, you should be able to access Prometheus on your host machine at `localhost:9090` and the metrics exposed by the Go application at `localhost:2112/metrics`.

Check and ensure that the job_name is registered. 

<p align="center">
<img width="1510" alt="prom_config" src="https://user-images.githubusercontent.com/2548160/273732084-60e7b5fa-d165-4740-ac64-10acc8636c62.png">
</p>

## Create TCP Chaos: Testing It All Out

To put the eBPF program and Prometheus monitoring into action, you need to introduce network issues in the VM. The `tc` (traffic control) command in Linux allows you to simulate network issues for testing purposes. The `create_tcp_chaos.sh` shell script below automates this process, first creating the chaos and then removing it after the test.


### The create_tcp_chaos.sh Shell Script

source : [create_tcp_chaos.sh](https://github.com/iogbole/ebpf-network-viz/blob/main/create_tcp_chaos.sh)

Here's the script that introduces packet loss and latency to `eth0` using `tc`.

```cpp
#!/bin/bash

# Define websites to send requests to.
websites=("http://example.com" "https://www.google.com" "https://www.wikipedia.org")
# Set the number of iterations for the loop.
loop_count=20
# Introduce network latency and packet loss using tc.
sudo tc qdisc add dev eth0 root netem loss 5% delay 100ms

# Loop to send requests to the websites.

for ((i = 1; i <= loop_count; i++)); do
    for site in "${websites[@]}"; do
        echo "Sending request to $site (iteration $i)"
        curl -sS "$site" > /dev/null  # s for silent and S for showing errors if they occur.
        sleep 1  # Wait for a second.
        wget -O- "$site" > /dev/null  # O- redirects output to stdout, as we don't want to save the file.

    done
done

# Remove the traffic control rule.

sudo tc qdisc del dev eth0 root
```

Run the script, and you should be able to observe the effects on your Prometheus metrics. Remember to execute the script with appropriate permissions.


## Grab a coffee: Reap the Rewards

Head over to your Prometheus interface and type tcp_retransmissions_total into the query bar. Switch to the graph view and marvel at the results of your hard work.

You're now in a position to set up alerts for TCP retransmissions. A common benchmark to consider is that a retransmission rate of 2% or greater generally indicates network issues that warrant attention.

So grab a cup of coffee, sit back, and enjoy the fruit of your labour!


<p align="center">
<img width="1510"  src="https://user-images.githubusercontent.com/2548160/273732219-e4b7bcf0-5d4a-456a-8197-543ecbcea061.png">
</p>


# Refs

* [Must read - Linux Man Page](https://www.man7.org/linux/man-pages/man2/bpf.2.html)
* [TCP retrans fields](https://github.com/iovisor/bcc/blob/master/tools/tcpretrans_example.txt)
* [BPF CORE](https://facebookmicrosites.github.io/bpf/blog/2020/02/19/bpf-portability-and-co-re.html)
* [TCP tracepoints](https://www.brendangregg.com/blog/2018-03-22/tcp-tracepoints.html)
* [eBPF applications](https://ebpf.io/applications/)