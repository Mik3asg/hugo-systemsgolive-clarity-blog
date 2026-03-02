---
title: "Pi-hole and Unbound on Raspberry Pi 4 – A Self-Hosted Ad-Blocking and Private DNS Resolver for a Tracker-Free Home Network"
date: 2026-03-01T12:19:06Z
draft: false
tags: ["Pi-hole", "Unbound", "DNS", "Raspberry Pi", "Homelab", "Ad Blocking", "Privacy", "DNSSEC", "Recursive DNS", "Self-hosted"]
categories: ["Homelab", "Networking", "Infrastructure"]
thumbnail: "images/pihole-unbound-raspberrypi-homelab.png"
summary: "Run Pi-hole and Unbound on a Raspberry Pi 4 to block ads and tracking domains across every device on your home network, while resolving DNS privately using a local recursive resolver — no third-party DNS provider required."
---

# Problem Statement

Every device on a home network performs DNS lookups before connecting to a website. DNS (Domain Name System) is the mechanism that translates a domain name into an IP address – and the operator of the DNS resolver can observe every domain queried.

In a typical home network, those queries are handled by an ISP or a public DNS provider such as Google or Cloudflare:

```
Device → ISP DNS / Public DNS → Internet
```

This means:

- Your ISP or DNS provider sees every domain queried
- All devices contribute to one central browsing history
- Ads and trackers resolve normally

DNS becomes a record of your activity.

---

# Goals / Non-Goals

**Goals**

- Block advertising and tracking domains across all devices on the network
- Resolve DNS privately without relying on a third-party provider
- Require no configuration on individual devices

**Non-Goals**

Installation is not covered. Refer to the official documentation:

- [Pi-hole Basic Install](https://docs.pi-hole.net/main/basic-install/)
- [Unbound Setup for Pi-hole](https://docs.pi-hole.net/guides/dns/unbound/)

---

# Solution

Two services running on a Raspberry Pi 4 address the problem:

| Service | Role |
|---|---|
| Pi-hole | Blocks ads and trackers across all devices at the DNS level |
| Unbound | Resolves allowed domains privately – no third-party DNS provider involved |

---

# Architecture

The diagram below compares DNS query flows side by side – with Pi-hole and Unbound on the left, and without on the right – showing exactly where queries are intercepted, filtered, or exposed at each step.

![pihole-unbound-dns-flow-comparison](/diagrams/pihole-unbound-dns-flow-comparison.drawio.svg)

**With Pi-hole + Unbound**

```
Devices → Pi-hole (filter) → Unbound (resolver) → Internet DNS
```

**Without Pi-hole + Unbound**

```
Devices → ISP DNS / Public DNS → Internet
```

---

# How It Works

## Homelab Setup

I set up Pi-hole and Unbound on a Raspberry Pi 4, connected to my home broadband router. Pi-hole is assigned a static IP address (`192.168.1.143`) and acts as the DNS server for the entire network.

### Hardware

- Raspberry Pi 4 Model B – 4 GB RAM
- Raspberry Pi OS Lite 64-bit (headless)
- Static IP address: `192.168.1.143`

### Services

| Component | Address | Purpose |
|---|---|---|
| Pi-hole | 192.168.1.143:53 | Network DNS server |
| Unbound | 127.0.0.1:5335 | Local recursive resolver |

### Router Configuration

On my broadband router, I disabled DHCP and set the DNS Primary Server to Pi-hole's static IP:

| Setting | Value |
|---|---|
| DHCP | Disabled |
| DNS Primary Server | `192.168.1.143` |

With DHCP disabled on the router, Pi-hole takes over as the DHCP server. All devices on the network automatically receive `192.168.1.143` as their DNS server – no manual configuration required on client devices.

I also pointed the router's own DNS to Pi-hole, so router-originated queries are resolved locally.

### Unbound Binding

Unbound listens only on localhost:

```
127.0.0.1:5335
```

This means:

- It is not exposed to the network
- Only Pi-hole can query it
- External access is prevented

### Upstream DNS Configuration

All third-party upstream DNS providers are disabled in Pi-hole.

Only Unbound is configured:

```
127.0.0.1#5335
```

This ensures DNS queries are resolved locally.

---

## Role of Pi-hole

Pi-hole acts as a DNS filtering layer.

It maintains a database of known ad, tracker, and malicious domains called **gravity**. When any device on the network queries a blocked domain, Pi-hole returns `0.0.0.0` – the request is stopped and the ad server is never contacted.

It works at the DNS level, before any content is loaded. No browser extension. No per-device setup. Every device on the network is covered automatically.

### DNS Flow Comparison

Without Pi-hole:

```
Device → Public DNS → Ad domain → Content loads
```

With Pi-hole:

```
Device → Pi-hole → Domain blocked locally
```

### Blocklist Management

Pi-hole includes a default blocklist on installation:

```
https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts
```

This list provides a baseline set of advertising and tracking domains.

I added additional lists from:

```
https://firebog.net
```

I selected only the recommended (green) lists to balance effective blocking with minimal false positives.

Update or rebuild the gravity database using:

```bash
pihole -g
```

This command downloads updated blocklists and rebuilds Pi-hole's filtering database.

### Pi-hole Dashboard

I access the dashboard at:

```
http://192.168.1.143/admin
```

The dashboard provides:

- Real-time DNS query logs
- Top queried and blocked domains
- Blocking statistics
- Device activity overview

![Pi-hole Dashboard](/images/pihole-dashboard.png)

---

## Limitation of Pi-hole Alone

Pi-hole blocks unwanted domains but does not resolve allowed domains itself.

Without Unbound:

```
Device → Pi-hole → Public DNS resolver
```

DNS queries are still visible to external DNS providers.

---

## What Is Unbound?

Unbound is a DNS recursive resolver running locally.

It performs the same function as public DNS services but operates within the home network.

Instead of forwarding queries to an external resolver, Unbound resolves domains directly using the DNS infrastructure.

Simple definition:

> A recursive resolver performs the complete DNS lookup process on behalf of a client.

---

## Public DNS vs Recursive Resolver

These terms are often confused.

| Term | Meaning |
|---|---|
| Recursive resolver | A DNS function or role |
| Public DNS | A resolver operated by a third party |

Google DNS, Cloudflare DNS, and Unbound all perform recursive resolution.
The difference is who operates the resolver.

---

## DNS Resolution with Unbound

The DNS system is organised as a hierarchy. Every domain name maps to a position within this tree:

```
                    . (root)
                    │
       ┌────────────┼────────────┐
      .org          .com         .uk      ← TLD (Top-Level Domain)
       │             │            │
 wikipedia.org   google.com    bbc.co.uk  ← second-level domain
       │
 en.wikipedia.org                         ← subdomain
```

When a device requests `en.wikipedia.org`, Unbound performs recursive resolution by locating the answer step by step:

1. Queries the **root servers** to determine who manages `.org`.
2. Queries the **.org TLD servers** to determine who manages `wikipedia.org`.
3. Queries the **authoritative servers** for `wikipedia.org` – the servers that hold the domain's official DNS records – to obtain the IP address for `en.wikipedia.org`.
4. Returns the result to Pi-hole, which then provides the answer to the requesting device.

This resolution is performed locally by Unbound, which queries the authoritative DNS system directly instead of forwarding requests to a public DNS resolver.

Public DNS providers such as Google or Cloudflare perform the same recursive process on external infrastructure. Because DNS queries are sent to their resolvers first, those providers can observe and potentially log the domains being requested.

With Unbound running locally, DNS lookups are handled by your own resolver, so no single third-party provider receives a complete history of DNS activity. Unbound also caches responses locally, allowing repeated queries to be answered more quickly and improving overall lookup performance.

---

## Behaviour Without Unbound

**Pi-hole Only**

```
Device → Pi-hole → Public DNS → Internet
```

Advertising domains may be blocked, but DNS queries remain externally visible.

**Pi-hole with Unbound**

```
Device → Pi-hole → Unbound → DNS hierarchy
```

DNS resolution occurs locally without reliance on public resolvers.

---

## DNSSEC – Validation

DNSSEC adds cryptographic signatures to DNS responses.

Unbound validates these signatures to confirm that responses originate from the legitimate domain authority.

If validation fails, the response is rejected.

DNSSEC protects against tampered or forged DNS responses.

### Verifying DNSSEC Validation

I tested DNSSEC validation directly against Unbound by running:

```bash
dig fail01.dnssec.works @127.0.0.1 -p 5335
dig +ad dnssec.works @127.0.0.1 -p 5335
```

**Invalid DNSSEC domain**

`fail01.dnssec.works` → `SERVFAIL`

Unbound correctly rejects the response because DNSSEC validation fails.

**Valid DNSSEC domain**

`dnssec.works` → `NOERROR` with `ad` flag

- `NOERROR` indicates a successful lookup.
- The `ad` flag (Authentic Data) confirms the response was cryptographically validated using DNSSEC.

Summary:

- `fail01.dnssec.works` → rejected invalid DNSSEC data
- `dnssec.works` → validated and authenticated successfully

---

## Privacy vs Encryption

These concepts address different concerns.

| Concept | Description |
|---|---|
| Privacy | Who receives DNS queries |
| Encryption | Whether queries are hidden during transmission |

Unbound improves privacy by removing centralised DNS providers.

DNS queries themselves remain standard DNS traffic.

An ISP may observe traffic in transit, but no single resolver receives a complete query history.

---

## Confirming the Setup

From a Windows device on my network, I ran:

```bash
nslookup en.wikipedia.org
```

This is what I got:

```
Server:  pi.hole
Address:  192.168.1.143

Non-authoritative answer:
Name:    dyna.wikimedia.org
Addresses: 185.15.59.224
Aliases: en.wikipedia.org
```

Interpretation:

- `Server: pi.hole` indicates Pi-hole handled the query.
- The request was not sent directly to an ISP or public DNS provider.
- Pi-hole forwarded the request internally to Unbound.

### Verifying Pi-hole Uses Unbound as Upstream

I ran the following tests to confirm that Pi-hole forwards queries to Unbound and that caching is working correctly.

#### First Query – Recursive Resolution

I ran:

```bash
dig fifa.com @127.0.0.1
sudo tail /var/log/pihole/pihole.log | grep "fifa"
```

What I observed:

- Query time ≈ 135 ms
- Pi-hole log showed:

```
forwarded fifa.com to 127.0.0.1#5335
reply fifa.com is 2.19.248.207
reply fifa.com is 2.19.248.224
```

Explanation:

- `127.0.0.1#5335` is Unbound.
- Pi-hole forwarded the request internally.
- Unbound performed recursive resolution by querying root, TLD, and authoritative DNS servers.
- The higher query time occurs because the result was not yet cached.

> If you see queries forwarded to `127.0.0.1#5335`, Pi-hole is correctly using Unbound as its upstream resolver.

![Pi-hole first query – recursive resolution](/images/phole-unbound-first-query-recursive-resolution.png)

#### Second Query – Cached Lookup

I ran the same query again:

```bash
dig fifa.com @127.0.0.1
sudo tail -n 10000 /var/log/pihole/pihole.log | grep "fifa"
```

This time:

- Query time ≈ 0 ms
- The log showed cached responses:

```
cached fifa.com is 2.19.248.207
cached fifa.com is 2.19.248.224
```

Explanation:

- The result is now served from Unbound's cache.
- No external DNS queries are required.
- Subsequent lookups are significantly faster.

![Pi-hole second query – cached lookup](/images/phole-unbound-second-query-cached-lookup.png)

---

# Alternative Approaches

DNS over HTTPS (DoH) and DNS over TLS (DoT) are encryption-focused alternatives. Both encrypt DNS queries in transit, preventing ISP-level interception.

| Approach | Encrypts DNS in transit | External resolver required |
|---|---|---|
| Default DNS | No | Yes – ISP or public provider |
| DoH / DoT | Yes | Yes – single provider (e.g. Cloudflare) |
| Pi-hole + Unbound | No | No – resolved locally |

The trade-off is centralisation. DoH and DoT hide the content of DNS queries from the network path, but they route all traffic through a single provider, which then receives a complete picture of every domain queried across the network.

This setup takes a different position: DNS queries are not encrypted in transit, but no single external resolver ever sees the full query history. Queries are distributed across the global DNS hierarchy rather than aggregated by one provider.

---

# Conclusion

The DNS stack operates entirely within the home network:

- **Router** → DHCP disabled; DNS Primary set to `192.168.1.143`
- **Pi-hole** → acts as DHCP server; blocks advertising and tracking domains
- **Unbound** → performs recursive DNS resolution locally
- **DNSSEC** → validates DNS responses cryptographically

**Pi-hole**

- Blocks advertising and tracking domains
- Operates across all network devices without per-device configuration
- Prevents unwanted connections before they occur

**Unbound**

- Performs recursive DNS resolution locally
- Removes reliance on public DNS providers
- Distributes DNS queries across the global DNS system

Result:

- All devices use Pi-hole automatically
- Ads and trackers are blocked at the DNS level
- No third-party recursive DNS provider receives query history
- DNS responses are validated and cached locally

Running Pi-hole and Unbound on a Raspberry Pi 4 was a deliberate homelab choice — privacy over encryption, with DNS resolution kept entirely in-house and no third-party provider involved at any step. No centralised resolver. No browsing history logged externally. No ads reaching any device on the network.
