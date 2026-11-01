# postgres-ha-lab
Production-style PostgreSQL 17 High-Availability lab configs (Patroni, etcd, HAProxy, Keepalived) and failure-testing documentation


<img width="1024" height="559" alt="image" src="https://github.com/user-attachments/assets/da093650-d305-4187-94e1-b3dfdcbf8f42" />

Architecture Overview

The lab simulates a resilient, self-healing database topology distributed across 5 virtual machines, eliminating single points of failure (SPOF) across the application, routing, and database layers.

Topology Diagram
(Reference the architecture diagram generated above for visual mapping)

Client Access Layer: Applications connect to a single stable Virtual IP (192.168.1.200) managed via VRRP.

Load Balancing Layer (HAProxy + Keepalived): Two redundant load balancer nodes run Keepalived for high availability of the VIP. HAProxy routes incoming traffic based on health checks queried from the Patroni REST API:

Port 15432: Routes exclusively to the active Primary (Leader) for read/write transactions.

Port 15433: Load-balances read traffic across available Standby replicas.

Database & Consensus Layer (Patroni + etcd):

3 PostgreSQL 17 database instances managed by Patroni for automated failover and switchover.

An underlying 3-node etcd distributed consensus store acts as the Distributed Configuration Store (DCS) for cluster state tracking and leader leasing.


postgres-ha-lab/
├── README.md                 # Architecture documentation & test runbooks
├── patroni/
│   └── patroni.yml           # Sanitized Patroni node configuration template
├── etcd/
│   └── etcd.yml              # Sanitized etcd cluster consensus config
├── load-balancer/
│   ├── haproxy.cfg           # HAProxy routing rules (ports 15432 / 15433)
│   └── keepalived.conf       # Keepalived VRRP and Floating VIP configuration
└── docs/
    └── failure-tests.md      # Detailed notes on crash simulation and timeline recovery
