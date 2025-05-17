## Summary

We’ve enhanced the existing three‑server LAMP infrastructure by introducing **three firewalls** for network segmentation and perimeter defense, an **SSL/TLS certificate** to serve encrypted HTTPS traffic for `www.foobar.com`, and **three monitoring agents** (e.g., Sumo Logic Installed Collectors) on each server to gather logs and metrics. Firewalls enforce traffic rules and isolate network zones; HTTPS ensures confidentiality, integrity, and authentication of user connections; and monitoring agents continuously collect data—logs, metrics, and traces—for real‑time observability. We’ll also cover how to track Queries Per Second (QPS) on your web servers and discuss residual architectural issues, including SSL termination at the load balancer, single‑primary database writes, and risks of colocating identical stacks on each node.

---
```mermaid
%%{init: {"flowchart": {"htmlLabels": false}}}%%
flowchart LR
  %% Internet to Edge Firewall
  A["User Browser"] -->|HTTPS (443)| FW1["⛨ Edge Firewall"]
  
  %% Edge Firewall to Load Balancer
  FW1 -->|HTTPS (443)<br/>SSL Cert Installed| LB["🔀 HAProxy Load Balancer"]
  
  %% Load Balancer to Internal Firewall
  LB --> FW2["⛨ Internal Firewall"]
  
  %% Internal Firewall to App Servers
  FW2 --> S1["App Server 1<br/>Nginx + App Runtime"]
  FW2 --> S2["App Server 2<br/>Nginx + App Runtime"]
  
  %% Each App Server to Database Firewall
  S1 --> FW3["⛨ DB Firewall"]
  S2 --> FW3
  
  %% DB Firewall to Database Nodes
  FW3 --> DB1["MySQL Primary"]
  FW3 --> DB2["MySQL Replica"]
  
  %% Replication Link
  DB1 -.->|Async Replication| DB2
  
  %% Monitoring Agents
  subgraph Monitors
    M1["Agent on LB"]
    M2["Agent on App Server 1"]
    M3["Agent on App Server 2"]
  end
  
  LB --> M1
  S1 --> M2
  S2 --> M3
```
---

## Additional Infrastructure Components

### 1. Three Firewalls

* **Why**: To segment the network into secure zones (e.g., DMZ, application tier, database tier), minimizing lateral movement by attackers.
* **What**: At least one firewall at the edge (between Internet and LB) and one between each application server tier and the database tier.

### 2. SSL/TLS Certificate

* **Why**: To encrypt all traffic between clients and `www.foobar.com`, protecting user data from eavesdropping and ensuring authenticity via a trusted CA-issued certificate.

### 3. Three Monitoring Agents

* **Why**: To provide observability into server health, application performance, and security events. Agents send logs, metrics, and traces to a SaaS (e.g., Sumo Logic) for analysis and alerting.

---

## Component Roles & Operation

### Firewalls

* **Purpose**: Inspect and filter incoming/outgoing packets based on policy rules to block unauthorized access and malware.
* **Placement**:

  * **Edge Firewall**: Protects against Internet threats.
  * **Internal Firewalls**: Enforce segmentation between LB, app servers, and database nodes.

### HTTPS Encryption

* **Why HTTPS**: Encrypts data in transit (TLS), thwarts man‑in‑the‑middle attacks, and verifies server identity to clients.
* **Certificate**: Deployed on the load balancer (or LB+app servers) for `www.foobar.com`, served on port 443.

### Monitoring

* **What It’s For**: Real‑time visibility into logs, metrics (CPU, memory, disk I/O), and traces for performance tuning and incident response.
* **Data Collection**: Installed agents (e.g., Sumo Logic Installed Collector) run on each server, compress/encrypt data, and push it to the cloud platform over HTTPS.

### QPS Monitoring

* **Definition**: Queries Per Second measures how many requests or database queries a server processes per second.
* **How to Monitor**:

  * **Agent Metrics**: Use application server or Nginx status modules to expose QPS metrics (e.g., `/status`), scraped by agents.
  * **Log Parsing**: Monitoring agents parse access logs to count requests over time.
  * **Threshold Alerts**: Configure alerts if QPS exceeds capacity, indicating need for scaling or investigation.

---

## Remaining Issues & Considerations

### 1. SSL Termination at Load Balancer

* **Issue**: Traffic between LB and backend servers may be unencrypted, violating zero‑trust principles and exposing data on internal networks.

### 2. Single MySQL Write Primary

* **Issue**: Only one node accepts writes; if it fails, no writes can occur, and failover may be manual or slow. Also replication lag can cause stale reads.

### 3. Homogeneous Servers with Full Stacks

* **Issue**: Running web server, app server, and database on each node complicates security hardening (larger attack surface), resource contention, and maintenance (patching multiple roles on each machine).

---

**Next Steps**:

1. **Enable end‑to‑end TLS** between LB and backends or deploy certificates on each app server.
2. **Promote a replica** to primary automatically upon failure or adopt multi‑primary clustering (e.g., MySQL Group Replication).
3. **Separate roles** onto dedicated servers or containers for better isolation.
4. **Implement SIEM** integration, intrusion detection, and alerting for a robust security posture.
