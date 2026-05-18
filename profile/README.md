<div align="center">
  <img src="https://edgeon.dev/logo.png" width="180" alt="EdgeOn Logo" />

  <h1>EdgeOn</h1>

  <p><strong>Next-Generation, High-Performance Edge Cloud & CDN Infrastructure</strong></p>

  <p>
    <img src="https://img.shields.io/badge/status-active-brightgreen?style=flat-square" />
    <img src="https://img.shields.io/badge/transport-QUIC-blue?style=flat-square" />
    <img src="https://img.shields.io/badge/load_balancing-geo--aware-orange?style=flat-square" />
    <img src="https://img.shields.io/badge/license-MIT-lightgrey?style=flat-square" />
  </p>
</div>

---

## What is EdgeOn?

EdgeOn is a full-stack traffic management platform. You register your domain, point it to EdgeOn's nameservers, and from that moment on — your DNS, routing, proxying, and load balancing are entirely under your control.

It combines:

- **EoDNS** — a high-performance authoritative DNS server with GeoDNS support
- **Proxy** — a geo-aware, health-conscious edge proxy layer
- **Reverse Proxy** — a flexible backend connector with 9 connection modes and a local dashboard
- **Health-based Load Balancer** — gradual, intelligent traffic shifting without hard cutoffs

Developers can use any subset of these components. The proxy and reverse proxy are **optional**.

---

## Architecture Overview

```
User's Browser / Device
        │
        │  DNS Query (port 53)
        ▼
┌───────────────────┐
│     EoDNS         │  ← Reads records from DB into RAM
│  (EdgeDNS)        │  ← GeoDNS: returns IP based on user's region
└───────────────────┘
        │
        │  If Proxy enabled → returns Proxy IP
        ▼
┌───────────────────┐
│   Edge Proxy      │  ← GeoDNS-aware routing
│                   │  ← Routes by Host header → correct socket
│                   │  ← Communicates with DNS for load shifting
└───────────────────┘
        │
        │  QUIC + Custom SSL (always)
        ▼
┌───────────────────┐
│  Reverse Proxy    │  ← 9 backend connection modes
│                   │  ← Injects health report headers
│                   │  ← Local dashboard for management
└───────────────────┘
        │
        ▼
   Your Backend(s)
```

---

## How It Works

### 1. DNS Resolution — EoDNS

When a visitor hits your domain, their device sends a DNS query to port 53 of the nameserver IP assigned by EdgeOn.

EoDNS:
- Loads all DNS records from the database **into RAM** for fast lookups
- Checks if **GeoDNS** is enabled for that domain
- If GeoDNS is active: detects the requester's region and returns the IP assigned to that region
- If the region has no specific IP mapped: falls back to the **default IP**
- If **Proxy** is enabled: returns the nearest proxy's IP instead, based on the configured GeoDNS group

### 2. GeoDNS Groups

You define GeoDNS groups from the dashboard. Each group maps regions to specific IPs or proxy nodes.

Example:
```
Group: "global-v1"
  US-East  → 1.2.3.4
  EU-West  → 5.6.7.8
  Default  → 9.10.11.12
```

The same group logic is shared between EoDNS and the Proxy layer for consistent routing end-to-end.

### 3. Edge Proxy

The proxy receives the request and:
- Re-checks GeoDNS to find the **nearest backend** for this visitor
- Routes using the **Host header** to match the correct socket / domain configuration
- Forwards to the Reverse Proxy over **QUIC** with a custom SSL certificate

### 4. Reverse Proxy

The reverse proxy supports **9 different backend connection modes** (plain TCP, TLS passthrough, Unix socket, etc.) and:
- Connects to your backend(s)
- Injects **health report headers** into the response (latency, error rate, load metrics)
- Is managed via a **local dashboard** (no cloud dependency)

### 5. Health-based Load Balancing

The Proxy reads the health headers injected by the Reverse Proxy and:
- Does **not** hard-cut unhealthy backends
- **Gradually shifts** traffic to healthier nodes — respecting GeoDNS proximity when possible
- Keeps load **evenly distributed** across all available backends
- Communicates with EoDNS to shift DNS-level routing when a proxy or region is overloaded

---

## Components

### EoDNS (EdgeDNS)

| Feature | Description |
|---|---|
| In-memory record store | All records loaded from DB into RAM |
| GeoDNS | Per-region IP mapping with fallback |
| Proxy-aware | Returns proxy IP when proxy mode is active |
| Port | UDP/TCP 53 |

### Edge Proxy

| Feature | Description |
|---|---|
| GeoDNS routing | Nearest-backend selection per visitor region |
| Host header routing | Maps requests to correct socket/backend |
| QUIC transport | All upstream connections use QUIC + custom SSL |
| DNS coordination | Signals EoDNS to rebalance under load |
| Health tracking | Reads and acts on health metrics per backend |

### Reverse Proxy

| Feature | Description |
|---|---|
| Backend modes | 9 connection types supported |
| Health headers | Injects metrics into responses for proxy consumption |
| Local dashboard | Domain & routing management, no external dependency |
| Transport to proxy | Always QUIC with custom SSL |

---

## Traffic Flow Example

```
1. alice.example.com → DNS query hits EoDNS
2. EoDNS checks: GeoDNS ON, Proxy ON, Alice is in Germany
3. EoDNS returns: EU proxy IP (from GeoDNS group)
4. Alice's browser connects to EU Proxy
5. Proxy checks GeoDNS → nearest backend: Frankfurt node
6. Proxy → Reverse Proxy (QUIC + custom SSL)
7. Reverse Proxy → Backend (Frankfurt)
8. Backend responds → Reverse Proxy injects health headers
9. Proxy reads health headers, strips them from Alice's response
10. Proxy records: Frankfurt health = OK
11. If Frankfurt degrades → traffic gradually shifts to Amsterdam
```

---

## Optional Components

EdgeOn is modular. You can use only what you need:

```
DNS only          → Use EoDNS standalone
DNS + Proxy       → Skip Reverse Proxy, connect directly to backend
Full stack        → EoDNS + Proxy + Reverse Proxy
```

Developers are **not required** to use the Proxy or Reverse Proxy layers.

---

## Getting Started

> Detailed setup guides are available in each component's repository.

**Quick overview:**

1. Register on [edgeon.dev](https://edgeon.dev)
2. Add your domain → receive nameserver records
3. Point your domain's NS records to EdgeOn's nameservers at your registrar
4. Configure DNS records (A, CNAME, etc.) from the dashboard
5. Optionally enable GeoDNS, Proxy, and/or Reverse Proxy per domain

---

## Key Design Decisions

**Why RAM-based record storage?**
DNS queries need sub-millisecond responses. Loading records into RAM eliminates DB round-trips on the hot path.

**Why QUIC between Proxy and Reverse Proxy?**
QUIC eliminates head-of-line blocking, handles connection migration, and performs better under packet loss — critical for edge-to-backend communication across regions.

**Why gradual traffic shifting instead of hard cutoffs?**
Hard cutoffs cause traffic spikes on surviving nodes. Gradual shifting keeps load balanced while giving degraded nodes time to recover, avoiding cascading failures.

**Why 9 backend connection modes?**
Different backends have different requirements — TLS termination, Unix sockets, plain HTTP, upstream proxies. The reverse proxy adapts to the backend, not the other way around.
