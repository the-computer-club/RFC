# luninet peer traversal proposal

**Author:** lunarix

**Revision:** 1, 2026

**Status:** Draft

---

## Abstract

luninet member sites may be located behind NATs with no static port forwarding
or publicly reachable endpoints. This document specifies a mechanism for
direct peer-to-peer WireGuard tunnel establishment between such sites using
UDP hole punching, DNS-Based Service Discovery (DNS-SD), and a lightweight
registry peer. No modifications to the WireGuard kernel implementation are
required.

---

## Purpose

This document defines the NAT traversal strategy for luninet. It is intended
to be read alongside the luninet network proposal and governs how member
peers that lack static endpoints discover each other and establish direct
tunnels. The hub-and-spoke fallback model (routing all traffic through a
central relay) is explicitly out of scope; direct peer paths are preferred
to reduce latency and avoid relay bottlenecks.

---

## 1. Background

### 1.1 Problem statement

WireGuard's crypto-key routing model does not require a statically configured
endpoint for a peer. However, when both endpoints of a desired tunnel are
behind NATs that neither operator controls, two problems arise:

- Neither peer can discover its own external IP:port autonomously.
- No signaling channel exists to communicate that information to the remote
  peer.

Standard WireGuard utilities resolve DNS hostnames for endpoint configuration,
but they have no mechanism for discovering UDP ports. This proposal addresses
both gaps without modifying WireGuard.

### 1.2 UDP hole punching

Because WireGuard operates over UDP, UDP hole punching is the traversal
technique employed. When a peer sends a UDP packet outbound through a NAT,
most NAT implementations create a mapping that permits inbound replies to
the same translated source IP:port. A third party that observes the translated
source IP:port can return traffic through this opening before the mapping
expires.

This technique is effective against full-cone and port-restricted cone NATs.
It does not reliably work against symmetric NATs; sites confirmed to sit
behind a symmetric NAT are out of scope for direct tunneling and must use
a relay.

---

## 2. Architecture

### 2.1 Roles

Three logical roles exist in the traversal system:

| Role     | Description |
|----------|-------------|
| Member   | A luninet site behind a NAT seeking a direct tunnel to another member. |
| Registry | A statically addressed, NAT-free node that observes member external endpoints and answers discovery queries. |
| Peer     | The remote member a given member wishes to reach directly. |

The Registry does not forward member-to-member traffic. Its sole functions
are endpoint observation (via an active WireGuard session with each member)
and endpoint advertisement (via DNS-SD).

### 2.2 Topology overview

Each member maintains a persistent WireGuard tunnel to the Registry peer.
This tunnel serves two purposes:

1. It opens and continuously refreshes a UDP mapping on the member's NAT,
   making the translated external IP:port observable by the Registry.
2. It provides a control-plane channel over which the member queries the
   Registry's DNS server for peer endpoint data.

Once a member has retrieved a peer's current external IP:port from the
Registry, it configures that peer's WireGuard endpoint locally and the
direct tunnel is established via the pre-existing NAT opening on each side.

---

## 3. Registry peer

### 3.1 WireGuard configuration

The Registry exposes a single WireGuard interface. Each member is provisioned
as a peer on this interface with only its tunnel address in `AllowedIPs`;
no default route or member-prefix routes are accepted. The Registry must
never route traffic between members.

Recommended keepalive on member side: 5 seconds. This is the interval at
which the NAT mapping observed by the Registry is refreshed.

### 3.2 Endpoint observation

The Registry learns each member's current external IP:port directly from the
WireGuard data plane — specifically from the source of the most recent
authenticated handshake or data packet from that peer. No additional protocol
is needed for this step.

---

## 4. Endpoint discovery via DNS-SD

### 4.1 Protocol choice

DNS-Based Service Discovery (RFC 6763) over DNS is used for endpoint
advertisement. DNS was chosen because:

- It is mature, cross-platform, and requires no custom wire protocol.
- It defines the SRV record type (RFC 2782) for advertising a host and
  port for a named service.
- Standard tooling (`dig`, `nslookup`) is sufficient for debugging.
- It operates naturally over the existing WireGuard tunnel to the Registry.

### 4.2 Service name

All WireGuard peers are advertised under the service type `_wireguard._udp`
within the luninet internal zone `luni.` (see luninet network proposal §3).

### 4.3 Public key encoding

WireGuard public keys are Base64-encoded 32-byte values. DNS labels are
case-insensitive (RFC 4343), making Base64 unsuitable as a record name.
Public keys must be re-encoded in Base32 (RFC 4648 §6) before use as
DNS labels. Base32 produces a 56-character, case-insensitive string.

Conversion at the command line:

```
$ cat pub.txt | base64 -d | base32
```


### 4.4 Record structure

For a member with Base32 public key `<KEY>` and external endpoint
`<IP>:<PORT>`, the Registry DNS zone contains:

```
_wireguard._udp.luni.          IN PTR   <KEY>._wireguard._udp.luni.
<KEY>._wireguard._udp.luni.    IN SRV   0 0 <PORT> <KEY>.luni.
<KEY>.luni.                    IN A     <IP>
```


IPv6 external addresses are represented with an AAAA record in place of A.

### 4.5 CoreDNS plugin (`wgsd`)

The Registry runs CoreDNS with the `wgsd` external plugin
(`github.com/jwhited/wgsd`). This plugin satisfies DNS-SD queries by
reading live peer state directly from the WireGuard interface via the kernel
API; no static zone files are maintained.

Minimum Corefile:
```
.:53 {
  wgsd luni. <wg-interface>
}
```

The DNS service must only be reachable via the WireGuard tunnel interface.
It must not be exposed on the Registry's public-facing interface.

---

## 5. Client operation (`wgsd-client`)

### 5.1 Responsibilities

Each member runs `wgsd-client`. On each invocation it:

1. Enumerates all configured WireGuard peers on the member's interface.
2. For each peer, queries the Registry DNS server for a PTR record under
   `_wireguard._udp.luni.` matching that peer's Base32 public key.
3. If an SRV record is returned, extracts the IP and port.
4. If the resolved endpoint differs from the currently configured endpoint,
   applies the update via `wg set`.

### 5.2 Scheduling

`wgsd-client` is not a daemon; it performs one pass and exits. It should
be invoked periodically via cron or a systemd timer. A polling interval
of 30 seconds is recommended as a starting point. Members with frequently
changing NAT mappings may require a shorter interval.

### 5.3 Registry peer handling

The Registry's own public key will appear in the member's peer list.
`wgsd-client` will attempt a lookup and receive no SRV record (the Registry
has a static endpoint). This is expected and non-fatal; the client logs
the miss and continues.

---

## 6. Keepalive requirements

Persistent keepalives are required on every member-to-Registry session and
on every direct member-to-member session. Without keepalives, NAT mappings
will expire and the hole punched for a direct session will close. The
recommended value is 5 seconds; operators must not set a value higher than
25 seconds.

---

## 7. Addressing

The Registry and each member require a dedicated tunnel address for the
traversal overlay. These addresses are drawn from the Management prefix
(`172.29.64.0/21`) defined in the luninet network proposal.

| Host     | Tunnel Address      |
|----------|---------------------|
| Registry | `172.29.64.1/32`    |
| Members  | `172.29.64.x/32`    |

Each member's WireGuard configuration must set `AllowedIPs = 172.29.64.1/32`
for the Registry peer and `AllowedIPs = <member-tunnel-addr>/32` for each
direct peer. No broader prefixes are permitted in the traversal interface's
allowed-IPs.

---

## 8. Policy summary

| Rule                                                   | Action       |
|--------------------------------------------------------|--------------|
| Registry DNS reachable on public interface             | Not permitted |
| Registry forwarding traffic between members            | Not permitted |
| Member-to-Registry keepalive interval > 25 s           | Not permitted |
| Symmetric NAT sites attempting direct tunnel           | Not supported |
| WireGuard source modified or patched for traversal     | Not permitted |
| Base64 public keys used directly as DNS labels         | Reject        |
| Traversal interface AllowedIPs wider than /32 per peer | Reject        |

---

## 9. References

- Jordan Whited, *WireGuard Endpoint Discovery and NAT Traversal using DNS-SD*,
  2020. https://www.jordanwhited.com/posts/wireguard-endpoint-discovery-nat-traversal/
- RFC 5389 — Session Traversal Utilities for NAT (STUN)
- RFC 6763 — DNS-Based Service Discovery
- RFC 2782 — DNS SRV Records
- RFC 4648 — Base16, Base32, Base64 Data Encodings
- RFC 4343 — Domain Name System (DNS) Case Insensitivity
- `wgsd` CoreDNS plugin: https://github.com/jwhited/wgsd
