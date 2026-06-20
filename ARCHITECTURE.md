# Architecture

## Objective

Add network-wide DNS-based ad/tracker filtering to a home lab, covering every
device on the LAN without per-device configuration — while preserving
graceful degradation if the filtering host goes down. The Pi already runs
several other services (file storage, a couple of self-hosted apps), so this
was added to existing infrastructure rather than standing up new hardware.

## Core concept: resolver vs. filter

A plain DNS resolver (e.g. a public resolver set as the router's primary DNS)
only translates domain names to IP addresses. It has no concept of "this
domain is an ad/tracker" and no mechanism to refuse an answer — that's not a
missing feature, it's simply outside what a resolver does.

Pi-hole is not a resolver. It's a **forwarder with a filtering layer in
front of it**:

1. Every DNS query from a LAN device hits Pi-hole first.
2. Pi-hole checks the query against a blocklist.
3. If matched, Pi-hole answers itself with a dead-end response — no real
   lookup happens, no redirect, just a non-routable result.
4. If not matched, Pi-hole forwards the query to a real upstream resolver
   (configured separately) and passes the answer back unchanged.

This distinction matters because "my public DNS provider hasn't blocked
this site yet" is the wrong mental model entirely — a public resolver was
never going to filter anything; that was never its job.

```
  LAN device                Filtering host              Upstream resolver
  (laptop, phone,           (Pi-hole + blocklist)        (public, DNS-secured)
   smart TV, etc.)
       │
       │  1. DNS query
       │  "ads.example.com"
       ▼
  ┌─────────────┐
  │  Filtering   │
  │    host      │
  └──────┬───────┘
         │
         ▼
   Matches blocklist? ──── yes ──▶  Respond locally: 0.0.0.0 / NXDOMAIN
         │                          (dead end — no real lookup happens)
         │ no
         ▼
  ┌──────────────────┐
  │ Forward query to  │ ───────▶  Real DNS lookup happens here
  │ upstream resolver │ ◀───────  Answer passed straight back, unchanged
  └────────┬──────────┘
           │
           ▼
     Answer returned to LAN device
```

Every query takes one of exactly two paths: blocked locally with no real
lookup, or forwarded untouched to a real resolver. There's no third path
and no partial filtering — the decision happens once, at the filtering
host, before anything leaves the network.

## Failure domains — the part that actually matters

Making Pi-hole the router's primary DNS means the whole LAN's name
resolution depends on one device. There are two genuinely different ways
that can fail, and they need two different fixes:

**The filtering process crashes, but the host is fine.**
Handled by the process supervisor (systemd on this stack) — the same
mechanism already supervising other services on the box. Supervisors detect
the crash and restart the process within seconds. No external monitoring
needed for this case.

**The whole host dies or hangs** (power loss, SD card failure, kernel
panic). No software running *on* the dying machine can save you here — by
definition, nothing on a dead host can act. The correct fix lives at the
router: a **secondary DNS entry pointing to a real public resolver**, not a
second instance of the filtering host.

This second point is the one that's easy to get wrong. A second Pi-hole
sounds like redundancy, but if it shares the same power circuit, same
storage medium, or same physical host as the primary, it shares the same
failure domain — it goes down for the same reasons the primary does, at
roughly the same time. True redundancy means the backup fails independently
of the primary, for unrelated reasons. A public resolver as the fallback
satisfies that; a second local Pi-hole doesn't.

**Net result when the failover is set up correctly:** the primary host
dying means ad-blocking silently pauses and internet access keeps working,
within seconds, at the OS resolver level on every client — no manual
intervention, no one on the network even needs to notice.

```
  Normal operation                    Filtering host is down
  ──────────────────                  ───────────────────────

  Router DNS settings:                Router DNS settings:
    DNS1 → Filtering host (primary)     DNS1 → Filtering host  ✗ no response
    DNS2 → Public resolver (backup)     DNS2 → Public resolver (backup)

       LAN device                          LAN device
           │                                   │
           │ query                             │ query
           ▼                                   ▼
   ┌───────────────┐                   ┌───────────────┐
   │ Filtering host │  ◀── answers      │ Filtering host │  ✗ times out
   │  (DNS1, alive) │      normally     │  (DNS1, dead)  │  (within seconds)
   └───────────────┘                   └───────┬────────┘
                                                │  client OS automatically
                                                │  retries on DNS2
                                                ▼
                                        ┌────────────────┐
                                        │ Public resolver │  ◀── answers,
                                        │   (DNS2)        │      unfiltered
                                        └────────────────┘

  Result: ad/tracker filtering         Result: filtering paused,
          active                               internet still works
```

The failover isn't something this build configures or scripts — it's a
property of how every device's OS already handles DNS: if the primary
doesn't respond, it tries the secondary automatically. The only design
decision here is making sure DNS2 points somewhere that doesn't share a
power supply, storage medium, or host with DNS1 — otherwise "backup"
fails for the same reason the primary did, at the same time.

## Internal hostname ≠ internet-facing

A DNS record is just a name-to-IP mapping. Nothing about having a record
inherently exposes a service to the internet — reachability is a separate,
independently configured concern (in this stack, a tunnel/ingress
configuration). A hostname can point at a private LAN address and resolve
to something meaningless from outside the network, while still being a
convenient internal name to type instead of an IP.

This is why the filtering dashboard in this build uses a real subdomain
with a valid TLS certificate, but is **not internet-reachable** — the
hostname exists, the cert exists, but there's no public ingress path to it.
Internal-only by configuration, not by obscurity.

## Final stack

| Layer | Component | Notes |
|---|---|---|
| Hardware | Single-board computer, wired Ethernet, static LAN IP | Already running other home-lab services |
| DNS/filtering | Pi-hole (installed via official script, reviewed before execution) | See SECURITY-REVIEW.md |
| Pi-hole's upstream resolver | A filtering-capable public DNS provider (DNSSEC enabled, no client-subnet sharing) | Adds a second filtering pass for anything the local blocklist misses |
| Blocklist | A standard, widely-used unified hosts list | ~85k domains at install |
| Filtering host's own admin dashboard | Bound to loopback only, reverse-proxied | Not directly reachable from the LAN — see incident writeup |
| Web admin access | Reverse proxy over HTTPS, valid certificate, internal hostname only | No public ingress rule for this hostname |
| Router primary DNS | Filtering host's static IP | Live, serving the whole household |
| Router secondary DNS | A public, filtering-capable resolver (same content policy as the upstream above) | Failure-domain-independent fallback — see above |
| Router's "advertise own IP" setting | Disabled | Was on by default; left enabled it would have created an undeclared third DNS path bypassing the filter entirely |

## Privacy choices in the upstream resolver

Some public resolvers share part of a client's IP/subnet with upstream
infrastructure for geo-optimized routing (EDNS Client Subnet). That's a
privacy cost — location-adjacent data leaking passively on every DNS
lookup — for a marginal speed gain. The upstream resolver selected for this
build supports tamper-detection (DNSSEC) without that subnet-sharing
behavior, consistent with the goal of location data only being shared when
deliberately invoked by an app, not leaking continuously as a side effect
of DNS resolution.

## Query logging — a visibility change, not a new exposure

Enabling query logging means the filtering host records every query
(domain, timestamp, requesting device) locally, on hardware already inside
the network and under direct control. This doesn't create new exposure —
an upstream public resolver was already seeing 100% of this query data
already, with zero visibility back to the household. Logging converts
already-existing, already-opaque data into something locally visible and
useful for verifying the install actually works. Correct sequencing:
enable logging to verify, consider reducing it later once the system is
trusted — not the reverse, since disabling logging first removes the only
way to confirm anything is functioning correctly.
