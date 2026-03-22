# luninet genesis network proposal
 
**Author:** lunarix  
**Revision:** 2, 2026  
**Status:** Draft  
**Depends on:** [0001-luninet.md](./0001-luninet.md)
 
---
 
## Abstract
 
The genesis network is operated by lunarix. All base policy and addressing rules are inherited from the luninet RFC ([0001-luninet.md](./0001-luninet.md)). This document covers only genesis-specific configuration.
 
---
 
## Purpose
 
This document describes how to connect to the genesis network. All specifications not defined here defer to RFC 0001.
 
---
 
## 1. Quick reference
 
| Parameter | Value            |
|-----------|------------------|
| ASN       | `4211555452`     |
| Gateway   | `172.29.64.1`    |
| Network   | `172.29.80.0/24` |
 
**Route table:**
 
| Prefix                | Destination     |
|-----------------------|-----------------|
| `172.29.64.0/18`      | luninet         |
| `172.29.80.0/21`      | genesis (local) |
| `fd49:093b:2b68::/48` | luninet         |
| `0.0.0.0/0`           | Mullvad exit    |
| `::/0`                | Mullvad exit    |
 
---
 
## 2. IPv4 Addressing
 
### 2.1 Allocated block
 
genesis operates within `172.29.80.0/21` as assigned by RFC 0001.
 
| Prefix           | Purpose        |
|------------------|----------------|
| `172.29.80.0/24` | Router clients |
| `172.29.81.0/24` | Unassigned     |
| `172.29.82.0/24` | Unassigned     |
| `172.29.83.0/24` | Unassigned     |
| `172.29.84.0/24` | Unassigned     |
| `172.29.85.0/24` | Unassigned     |
| `172.29.86.0/24` | Unassigned     |
| `172.29.87.0/24` | Unassigned     |
 

## 3. IPv6 Addressing

Following the SLAAC model, generate a unique /64 address from the /48. 

| Prefix                  | Purpose        |
| `fd49:093b:2b68::1/128` | Router         |
| `fd49:093b:2b68:1::/56` | Router Clients |
