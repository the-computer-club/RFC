# luninet network proposal

**Author:** lunarix


**Revision:** 2, 2026


**Status:** Draft

---

## Abstract

luninet is a layer 3 VPN network for computer club members. It provides data transport and NAT traversal services across member sites. Although the address space is private, all addressing and routing policy should be treated with the same discipline as a public network.

---

## Purpose
This document acts as foundational spec that can be then in conjuction with new changes, and provides a starting point and standard that's delivered with little effort, and leaves room for expansion.

--- 


## 1. IPv4 Addressing

### 1.1 Allocated block

luninet controls `172.29.64.0/18`, covering `172.29.64.0` – `172.29.127.255`.
Currently with the assigned IPv6 Blocks:

| Prefix                | Purpose |
|-----------------------|---------|
| `fd49:093b:2b68::/48` | lunarix


Despite being RFC 1918 space, this block should be treated as if it were globally routed. Any packet sourced from or destined to an address outside this block (excluding legitimate router link addresses) must be dropped at the network edge.

### 1.2 Subnet allocation

| Prefix            | Purpose                                  |
|-------------------|------------------------------------------|
| `172.29.64.0/21`  | Management                               |
| `172.29.72.0/21`  | Reserved                                 |
| `172.29.80.0/21`  | lunarix                                  |
| `172.29.88.0/21`  | jeffrey                                  |
| `172.29.96.0/21`  | Unassigned                               |
| `172.29.104.0/21` | Unassigned                               |
| `172.29.112.0/21` | Unassigned                               |
| `172.29.120.0/21` | Unassigned                               |
| `172.29.255.0/24` | Router links (point-to-point, /31 pairs) |

IPv4 support is included for legacy application compatibility only. New services should prefer IPv6.

### 1.3 Router links

All router-to-router links are addressed from `172.29.255.0/24` using /31 pairs. These prefixes must never be redistributed into eBGP.

---

## 2. IPv6 Addressing

Each member router must generate a unique /48 ULA prefix (`fc00::/7`) and advertise it via eBGP. Inter-site traffic must be carried over IPv6; IPv4 inter-site routing is not permitted.

The network will route `::/0` and `0.0.0.0/0` internally but will only advertise `fc00::/7` to peers.

---

## 3. DNS

A top-level domain `.luni` is used internally.

| Name    | Type | Value                |
|---------|------|----------------------|
| `luni.` | A    | `172.29.64.1`        |
| `luni.` | AAAA | `fd49:093b:2b68::53` |

---

## 4. Routing Architecture

### 4.1 Internal routing

Each zone manages its own internal routing independently. Zone operators are responsible for route policy within their assigned prefix.

### 4.2 Dynamic route export

Each zone router must export its prefixes via eBGP to its upstream block router, which in turn advertises them to the core router at `172.29.64.1`.

### 4.3 External routing (inter-site)

- All inter-site sessions must run over IPv6.
- Only 32-bit private ASNs are accepted (`4200000000`–`4294967294`).
- Only `fc00::/7` (ULA) prefixes are advertised to external peers.
- Default routes (`0.0.0.0/0` and `::/0`) may be accepted and routed internally but must not be re-advertised.
- Router link prefixes (`172.29.255.0/24`) must be filtered from all eBGP advertisements.

---

## 5. Policy Summary

| Rule                                        | Action        |
|---------------------------------------------|---------------|
| Source/destination outside `172.29.64.0/18` | Drop (IPv4)   |
| Router link prefixes in eBGP                | Reject        |
| IPv4 inter-site routing                     | Not permitted |
| ASN outside 32-bit private range            | Reject        |
| Advertisement of non-ULA IPv6 to peers      | Not permitted |


