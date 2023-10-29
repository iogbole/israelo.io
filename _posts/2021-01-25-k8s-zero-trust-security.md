---
layout: post
title:  "Implementing Zero Trust Security in Kubernetes"
author: israel
categories: [ 'Cloud Native' ]
tags: [ containers, kubernetes, application, developer, cloud-native, security ]
image: https://user-images.githubusercontent.com/2548160/105699638-96867e00-5eff-11eb-825f-b9d93cea87ad.png
date:   2021-01-25 11:10:35 +0300
description: "The vulnerability in castle-and-moat security systems is exacerbated by the emergence of micro-services and cloud-native technologies. The question is, how can a Zero trust model be implemented in a Kubernetes environment ?"

---

Zero trust security is an information technology security model that requires ascetic verification for everyone and everything trying to access resources on a private network, regardless of whether they are sitting within or outside of the network perimeter. There is no single specific technology associated with a zero trust architecture; it is a holistic cyber security methodology that combines various  principles and technology tenets.

Traditional network security is based on the military inspired castle-and-moat design. In a castle-and-moat security model, it is difficult to obtain access from outside the network, but by default, everyone inside the network is trusted. The problem with this approach is that once an attacker gains access to the network, they can freely control all content inside.

This vulnerability in castle-and-moat security systems is exacerbated by the emergence of micro-services and cloud-native technologies. Organisations no longer have their applications and data in just one place - behind a dedicated firewall. Cloud-native makes it more difficult to have a single security control for an entire network.

Zero trust is the foundation of a growing trend of security-first infrastructures, and it is one of the core philosophies of the  <a href="https://dodcio.defense.gov/Portals/0/Documents/DoD%20Enterprise%20DevSecOps%20Reference%20Design%20v1.0_Public%20Release.pdf" target="_blank"><b>DevSecOps</b></a> culture. The Zero trust model overturns the long-held traditional view that every entity (be it the application, server, networking software or hardware) found in an internal network could be trusted, not even the infrastructure itself should be trusted. This added layer of security has been shown to prevent data breaches.

According to a recent publication by the <a href ="https://securitybrief.eu/story/data-breaches-costing-companies-millions-could-incident-response-help" target="_blank"> securitybrief.eu </a>, data breaches cost companies $3.86 million per breach. The other none quantifiable cost that is often not talked about is reputation. Considering the consequences of a data breach to organisations, it should come as no surprise that many organisations are now eager to adopt a zero-trust security policy.

Moreso, since Kubernetes plays a massive role in enabling cloud-native deployments, the question is,  how can a Zero trust model be implemented in a Kubernetes environment? First, let's review the core principles of a Zero trust architecture, then see how they apply to Kubernetes.

## Principles of Zero Trust Architecture

Zero-trust architectures generally follow these principles:

- Authorisation grants should follow the principle of least privilege, allowing only the bare minimum permissions required for a client’s workload.

- Security controls should apply equally to all entities - including apps and infrastructure, regardless of their network location.

- Network connections should be authenticated at both ends, by the server and the client. Client authentication by the server is generally expected now, but clients should also verify that they have connected to a valid server.

- All network connections and transactions should be subject to continuous monitoring for analysis.

## Implementing Zero Security model in Kubernetes

In a containerised environment such as Kubernetes, Zero trust security must be baked into the design, build, development and runtime stages. Kubernetes defaults are generally not suited for production use as it comes with a wide range of liberal policies. Kubernetes policies need to be configured inline with the security and business objective.

Like most things with Kubernetes, there are several methods of implementing zero-trust principles within a Kubernetes cluster - with varying results. Zero Trust can be implemented in Kubernetes by using Network Policies (recommended for small workloads) and/or Service meshes. Network Policies may be sufficient for a small to medium-sized Kubernetes workloads.  Let's consider some of the ways to implement Zero trust in Kubernetes.

### Network Policy

Kubernetes, by default, allows all pods to communicate with each other and the internet. Network Policies can be used to implement zero-trust between pods by creating a deny-all policy that prevents inbound and outbound communications between pods by default.

```yaml 

apiVersion: networking.k8s.io/v1 
kind: NetworkPolicy 
metadata:   
  name: deny-all 
spec:   
  podSelector: {}  #Blank {} applies to all pods  
  policyTypes:   
  - Ingress   
  - Egress

```

The example policy shown above denies both ingress and egress traffics to/from all pods by default. Once this is in place, other policies can then be created to override the default deny-all Network Policy by explicitly defining pod labels allowed to communicate with each other. For example, the Network Policy below restricts ingress to `db` pod from ONLY the `api` and `web` pods respectively:


```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: db-allow-services
spec:
  podSelector:
    matchLabels:
      app: zenithcv
      role: db
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: zenithcv
          role: web
    - podSelector:
        matchLabels:
          app: zenithcv
          role: api
```

In other words, only pods whose labels match: (`app: zenithcv` and `role: db`) and (`app: zenithcv` and `role: api`) are allowed to connect to the database pod.

### Service Mesh

While Network Policy works brilliantly, it is difficult to scale. This is one of many reasons why Service meshes have emerged as a popular solution for many to achieve their Zero trust architecture’s goals. Service meshes create a virtual network layer to connect and control distributed application services. Most service mesh solutions did not initially focus on network security; they concentrated on facilitating and managing intelligent service discovery and request routing. However, the most popular open-source projects now provide features suitable for zero trust architecture. Service meshes are increasingly becoming popular; they eliminate many of the burdens of making significant changes to enable strict authentication and authorisation controls by creating overlays that do not require individual applications modification.

Kubernetes Service meshes such as HashiCorp Consul, Istio, Linkerd and the like typically use point-to-point, distributed routing mechanism, which creates each cluster pod proxy instance. These proxies can manage client TLS certificates, which the proxy can use to prove its identity when making connections to other services or receiving connections from other clients. The method of using TLS certificates to provide identity certification on the client and server is called Mutual Transport Layer Security (mTLS). In addition to performing connection authentication, mTLS is also used to encrypt network connections.
In addition to authentication and encryption, different service meshes support various authorisation sources, ranging from static lists to integrations with third-party single sign-on or other services.

While Service meshes offer several Zero Trust Security's core benefits, they not provide a <b> complete </b> zero-trust solution for Kubernetes clusters. Even if you cannot achieve a perfect zero-trust architecture in your Kubernetes clusters, any incremental changes you make in Service Mesh direction will help protect your cluster, and its workloads as most tools that are primarily built to provide Kubernetes Security leverages the mesh. This brings us to Portshift.

## Portshift

Portshift came into my radar when Cisco announced its acquisition.

<a href="https://www.portshift.io/" target="_blank"> Portshift </a> is a Kubernetes-Native Security Platform that provides a Single Pane of glass for Containers and Kubernetes. It encrypts Intra and iter cluster communications and provides security insights into CI/CD pipelines and Kubernetes runtime.

What distinguishes Portshift (in my opinion) is their agentless approach. I have had over 4 years experience in the Application Performance Monitoring (APM) industry, and I am well acquainted with the pains, overhead and constraints that agent-based implementation can cause, especially in a containerised environment.  The agentless approach means it is easy to scale Kubernetes deployments with Portshift. It also means that it is lightweight and has fewer footprints whilst protecting solutions Kubernetes deployments from threats and threats vulnerabilities across images, containers, runtime deployments and Kubernetes infrastructure.

## Summary

As applications and networks are becoming increasingly complex, so are security threats. The traditional security perimeter around organisations infrastructure, apps and data is no longer sufficient and does not work with cloud-native and micro-services architecture. Building a Zero trust model into architectures from the ground-up is the best approach for securing cloud-native infrastructure and workloads.

Kubernetes default Network Policies are porous and should not be used as-is in Production. Network Policies and Service Meshes are good entrypoints way to start exploring and implementing Zero Trust Security in Kubernetes, but they do not provide the full benefits of Zero Trust Security. Kubernetes-native security solutions such as Portshift may come handy. Portshift uses an agentless approach which makes it lightweigt and easy to scale.

## References:

- https://www.ciosummits.com/wp-zero-trust.pdf

- https://www.gartner.com/en/documents/3986053/market-guide-for-zero-trust-network-access

- https://www.portshift.io/product/kubernetes-security
