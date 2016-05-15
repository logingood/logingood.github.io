---
layout: post
title:  "Experiments with container networking: Part 3"
date:   2016-05-15 17:24:00 +1000
categories: kubernetes cni
---

# Kubernetes with BaGPipe BGP and CNI

The third part of CNI, Kubernetes and EVPN BGP discussion brings us to the solution that will look as on a topology that shown above. We want something that will stitch our Kubernetes-orchestrated datacenter networks together and provide us with seamless inter and intra DC communication between containers. All this should be implemented with regards to service-oriented model (micro-service architecture).

{% include image.html url="/images/Kubernetes-DC-BGP.png" description="Multi-datacenter Kubernetes network routed with BGP" %}

To proof that we can deliver this design we require to prepare a proof-of-conecpt environment. To do this we require a mean to integrated BaGPipe BGP solution inside Kubernetes environment. Likely kubelet (a process that responsible for managing containers on the Kubernetes node) support CNI interface. We can instruct it using configuration arguments to use our CNI plugin. 

# How CNI plugin talks to BaGPipe BGP

# Proof of concept BaGPipe BGP, CNI plugin and Kubernetes
