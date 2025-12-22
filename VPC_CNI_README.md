# VPC-CNI Explanation for EKS

## What is VPC-CNI?
Amazon EKS uses the AWS VPC CNI plugin to assign **real VPC IP addresses** to Kubernetes pods. This means:
- Pods communicate directly using VPC networking
- No overlay networking layer
- Lower latency and simpler routing

Every pod gets a native AWS VPC IP address from the node’s available Elastic Network Interface (ENI) secondary IPs.

---

## How Pod IP Capacity Is Determined
Pod IPs on each node depend on:
1. Number of ENIs supported by the EC2 instance type
2. Number of secondary IPs supported per ENI

### Formula:
```
(Max ENIs × Max IPs per ENI) – 1 = Maximum Pods Per Node
```
The node's primary IP is reserved for the node itself.

---

## ENI Simplified
ENI is like a network card.  
Example: A laptop may have Ethernet and Wi-Fi—it has multiple network interfaces.  
EC2 nodes have ENIs. Each ENI can hold multiple IP addresses. Pods receive those IPs.

---

## t3.medium Calculation (Correct)
A t3.medium instance supports:
- **3 ENIs**
- **6 IP addresses per ENI**

### Calculation:
```
3 ENIs × 6 IPs = 18
18 – 1 = 17 usable Pod IPs
```

So, a **t3.medium EC2 worker node can run up to 17 pods** using VPC-CNI.

---

## m5.xlarge Calculation (Correct)
An m5.xlarge instance supports:
- **4 ENIs**
- **30 IPs per ENI**

### Calculation:
```
4 ENIs × 30 IPs = 120
120 – 1 = 119 usable Pod IPs
```

So, an **m5.xlarge EC2 worker node can run up to 119 pods** using VPC-CNI.

---

## Final Short Summary
```
t3.medium = (3×6)–1 = 17 pods max
m5.xlarge = (4×30)–1 = 119 pods max
```

These pod counts are AWS hardware networking limits, not Kubernetes configurations.

---
