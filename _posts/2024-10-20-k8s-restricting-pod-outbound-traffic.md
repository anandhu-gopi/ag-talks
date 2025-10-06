---
layout: single
title: "K8s: Restricting Your Pod's Outbound Traffic 🚦"
date: 2024-10-20
author_profile: true
read_time: true
comments: true
share: true
categories: [DevOps]
tags: [kubernetes, egress, network-policies, security, networking]
excerpt: "Learn how to control outbound traffic from Kubernetes pods using egress rules and network policies for enhanced security."
---

![Egress Flow]({{ site.baseurl }}/assets/images/egress/flow.jpeg)

Have you ever encountered a scenario where you need to control outbound traffic from your Kubernetes pod? Specifically, how can you ensure that your pod only communicates with the services necessary for its operation? Today, I will discuss one way to control outbound traffic in Kubernetes using egress rules and network policies.

## 1️⃣ Ingress vs. Egress

Those who have worked in the K8s world are familiar with the term ingress. When you deploy an application in K8s and wish to expose it to the outside world, you typically create a deployment, service, and ingress.

While ingress manages incoming traffic, egress takes care of your outbound traffic. You can create network policies with egress rules to ensure that your application only communicates with the services necessary for its operation, thereby enhancing application security.

## 2️⃣ Egress, Network Policies, and Network Plugins

Just as we have ingress rules and ingress controllers implementing them, you can create network policies and include your egress rules there. Network plugins will take care of enforcing the egress rules (one such plugin is Calico). Please note that creating a NetworkPolicy resource without a controller that implements it will have no effect.

## 3️⃣ Targeted Egress Rules Using Pod Selectors 🎯

When creating network policies with egress rules, it's important to specify which deployments or pods the policy applies to. You can use a "podSelector" to target specific pods or leave it empty to apply the policy to all pods in the namespace.

## 4️⃣ Egress Rules: What Can Be Controlled?

You can apply the following restrictions to a pod via egress rules:

- **Pods** that your pod can communicate with.
- **Namespaces**: to access services running in a particular namespace.
- **IP blocks (CIDR ranges)** that your pod can connect to.

🚫 **When you create egress rules for your pod, it can only communicate with the specified pods, namespaces, or IP blocks in those rules; everything else is blocked.**

## 5️⃣ Example

I have attached an example of an egress rule where a front-end application is restricted to communicate only with a back-end application and kube-dns (for resolving DNS names)

![Egress YAML Example]({{ site.baseurl }}/assets/images/egress/yaml.jpeg)

---

## Acknowledgments

A huge thank you to my awesome colleague [Aswath PT](https://www.linkedin.com/in/aswath-pt/) from our DevOps team for introducing me to the topic of egress and network policies. 


[Original post from Linkedin](https://www.linkedin.com/posts/anandhu-gopi-691b35144_is-wasm-the-new-docker-a-few-days-ago-activity-7268649431535611904-MgnC)
