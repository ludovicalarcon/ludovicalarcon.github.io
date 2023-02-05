---
layout: post
title: "What is a Service Mesh?"
tags: ["Service Mesh", "101"]
---

A `Service Mesh` is a dedicated infrastructure layer that transparently add capabilities like observability, traffic management and security without the need to add them to your own code.  
It handles service-to-service communication by routing all inter-service communication through a proxy.

I hope this series of blog posts will help people in their service mesh journey.

- __What is a Service Mesh ?__
- Istio (Incoming)

## Key features

- Load balancing
- Observability
- Service discovery across environments
- mTLS encryption
- Automatic retries
- Granular traffic control and routing for HTTP and gRPC
- RBAC
- Authentication & Authorization

## Architecture

### ![](/assets/images/2023-02-05-servicemesh.png)
[Source: nginx service mesh architecture](https://www.nginx.com/wp-content/uploads/2019/02/service-mesh-generic-topology.png)

A service mesh is composed of 2 key components:
- A control plane where configurations are
- A data plane for each service that contains the instance (LB, routing, service discovery, encryption, auth, etc..) and a sidecar proxy

## Drawbacks

- Increase of number of instances and resources usage
- Communication adds an extra step (call first go through proxy)
- Service meshes don’t support integration with other systems or services
- Need the team to skill up on service mesh knowledges
- Add complexity on your architecture

## No need of Service Mesh

You don't need a service mesh if:
- You don't rely on microservices architecture
- You don't need all the feature provide by service mesh  
For example if you just want observability part, you can install the tools on your own without adding overhead and complexity with service mesh.  
- You don't have the time and/or the people to skill up on service mesh

## Principal offering

- [Linkerd](https://linkerd.io/docs/)
- [Istio](https://istio.io/latest/docs/)
- [Consul Connect](https://developer.hashicorp.com/consul/docs/connect)
- [Open Service Mesh (OSM)](https://docs.openservicemesh.io/)
- [Gloo Mesh](https://docs.solo.io/gloo-mesh-enterprise/latest/)
- etc...