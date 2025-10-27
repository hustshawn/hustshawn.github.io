---
title: "Hubble Functionality Guide"
date: 2025-10-27T15:48:47+08:00
draft: false
description: ""
tags: ["Cilium", "Hubble", "Architecture"]
categories: ["Observability", "Service Mesh"]
author: "Shawn Zhang"
showToc: true
TocOpen: false
hidemeta: false
comments: false
disableHLJS: false
disableShare: false
searchHidden: false
cover:
    image: "https://github.com/cilium/hubble/blob/main/Documentation/images/hubble_logo.png?raw=true"
    alt: ""
    caption: ""
    relative: false
    hidden: true
---

Hubble is Cilium's **observability platform** built on top of eBPF for network visibility and monitoring.

![Hubble Architecture](https://github.com/cilium/hubble/blob/main/Documentation/images/hubble_arch.png?raw=true)

## Components Overview

### 1. Hubble (in Cilium Agent)

**Location:** Runs inside each Cilium pod (DaemonSet on every node)

**Functionality:**
- **Captures network flows** using eBPF at the kernel level
- **Monitors all pod-to-pod traffic** on that node
- **Collects metadata**: source/dest IPs, ports, protocols, HTTP methods, DNS queries
- **Stores flows in memory** (ring buffer)
- **Exposes gRPC API** on port 4244 for querying flows

**What it sees:**
```
Pod A → Pod B (HTTP GET /api)
Pod C → External IP (TCP SYN)
Pod D ← DNS response
Network policy DROPPED packets
```

### 2. Hubble Relay

**Deployment:** Single pod aggregating cluster-wide data

**Functionality:**
- **Aggregates flows** from all Cilium agents across all nodes
- **Single query endpoint** - you query Relay, it queries all nodes
- **Provides cluster-wide view** of network traffic
- **Handles TLS** between itself and Cilium agents
- **Exposes gRPC API** on port 4245

**Architecture:**
```
hubble CLI/UI
      ↓
Hubble Relay (port 80/4245)
      ↓
   ┌──┴──┬──────┬──────┐
   ↓     ↓      ↓      ↓
Node1  Node2  Node3  Node4
(Cilium agents with Hubble on port 4244)
```

**Why it's needed:**
- Without Relay: You'd need to query each node individually
- With Relay: Single query gets flows from entire cluster

### 3. Hubble UI

**Deployment:** Single pod with 2 containers

**Containers:**
- **Frontend:** Web UI (React app)
- **Backend:** API server that talks to Hubble Relay

**Functionality:**
- **Visual service map** - Shows pods/services as nodes, traffic as edges
- **Real-time flow visualization** - Green (allowed), red (denied)
- **Filtering** - By namespace, pod, verdict, protocol
- **Flow details** - Click on connections to see packet details
- **Network policy visualization** - See what's allowed/blocked

**What you see:**
```
Service Map:
  frontend ──(green)──> backend
  backend  ──(green)──> database
  attacker ──(red X)──> backend (policy denied)
```

### 4. Hubble Services

**hubble-peer (ClusterIP:443)**
- Used by Hubble Relay to discover Cilium agents
- Peer service for node-to-node communication

**hubble-relay (ClusterIP:80)**
- Entry point for Hubble CLI and UI
- Aggregates data from all nodes

**hubble-ui (ClusterIP:80)**
- Web interface access point
- Serves the UI frontend

## Data Flow

```
1. Network packet arrives at node
         ↓
2. eBPF captures packet metadata (Cilium agent)
         ↓
3. Hubble (in Cilium) stores flow in memory
         ↓
4. Hubble Relay queries all Cilium agents
         ↓
5. Hubble UI/CLI queries Relay
         ↓
6. You see network flows!
```

## What Hubble Captures

### Layer 3/4
- Source/destination IPs and ports
- Protocols (TCP, UDP, ICMP)
- Packet verdicts (forwarded, dropped, denied)

### Layer 7 (Application)
- **HTTP:** methods, paths, status codes
- **DNS:** queries and responses
- **Kafka:** topics and messages
- **gRPC:** methods and status

### Security
- Network policy enforcement
- Identity-based access control
- Dropped/denied connections

### Performance
- Latency metrics
- Connection tracking
- Flow rates

## Use Cases

### 1. Troubleshooting
```bash
# Why can't Pod A reach Pod B?
hubble observe --from-pod default/podA --to-pod default/podB
```

### 2. Security Monitoring
```bash
# What's being blocked?
hubble observe --verdict DROPPED

# Watch denied traffic in real-time
hubble observe --verdict DROPPED --follow
```

### 3. Service Dependencies
```bash
# What does my frontend talk to?
hubble observe --from-pod default/frontend

# See all traffic in a namespace
hubble observe --namespace default
```

### 4. Compliance
- Network flow logs for auditing
- Policy enforcement verification
- Traffic pattern analysis

## Common Commands

```bash
# Check Hubble status
hubble status

# Watch all flows in real-time
hubble observe --follow

# Filter by namespace
hubble observe --namespace kube-system

# Filter by verdict
hubble observe --verdict FORWARDED
hubble observe --verdict DROPPED

# Filter by pod
hubble observe --from-pod default/frontend --to-pod default/backend

# Show last N flows
hubble observe --last 20

# Open Hubble UI
cilium hubble ui
```

## Component Summary

| Component | Purpose | You Interact With |
|-----------|---------|-------------------|
| Hubble (in Cilium) | Captures flows on each node | No (automatic) |
| Hubble Relay | Aggregates cluster-wide flows | Via CLI/UI |
| Hubble UI | Visual interface | Yes (browser) |
| hubble CLI | Command-line queries | Yes (terminal) |

## Key Benefits

✅ **No application changes** - eBPF captures at kernel level  
✅ **Low overhead** - Efficient eBPF programs  
✅ **Real-time** - See traffic as it happens  
✅ **Cluster-wide** - Single view across all nodes  
✅ **Deep visibility** - L3-L7 protocol awareness  
✅ **Security insights** - Policy enforcement visibility  
✅ **Troubleshooting** - Debug connectivity issues quickly  

## Configuration

Current Hubble settings can be viewed with:
```bash
cilium config view | grep -i hubble
```

Key settings:
- `enable-hubble: true` - Hubble enabled
- `hubble-listen-address: :4244` - gRPC API port
- `hubble-disable-tls: false` - TLS enabled for security

## Summary

Hubble gives you **network observability superpowers** - see every connection, understand traffic patterns, debug issues, and verify security policies, all without touching your applications! 🔍
