# Investigations

Two debugging stories from this project. One is a real incident with a real
root cause. The other looked like a real incident and wasn't — included
because correctly identifying a false alarm is as important a skill as
fixing a true one, and the process of ruling it out is the more interesting
part.

---

## Incident: a port conflict that silently broke an unrelated service

```
  Nginx already holds 0.0.0.0:80/443 (IPv4 wildcard)
            │
            ▼
  Filtering host installs ─── finds IPv4 taken ─── silently
                                                     binds IPv6 instead
            │                                       (no error, no warning)
            ▼
  Install looks 100% successful  ◀── but isn't
            │
            │  hours later
            ▼
  ┌─────────────────────────────────────────────┐
  │  Uptime monitor on unrelated app: 403 errors   │
  │  IPv6 clients silently served the filtering    │
  │  host's 403 page instead of the real app        │
  └─────────────────────────────────────────────┘
            │
            ▼
  Attempt 1: scope filtering host to loopback only (v4 + v6)
            │
            ▼
  ┌─────────────────────────────────────────────┐
  │  Webserver fails to bind ANYTHING               │
  │  No error in any obvious log                    │
  │  Real cause hidden until a debug flag is          │
  │  enabled — buried error: loopback already         │
  │  in use too (0.0.0.0 includes 127.0.0.1)            │
  └─────────────────────────────────────────────┘
            │
            ▼
  Root cause confirmed: 0.0.0.0 is a wildcard —
  it already includes loopback. Two processes can't
  both hold the same port via "wildcard" + "loopback".
            │
            ▼
  Real fix: move filtering host's webserver to its own
  dedicated port entirely (no collision possible).
  Verified bind, verified response, rebuilt proxy rule.
```

One root cause, two failed attempts, two separate debugging passes — the
first to find the conflict existed at all (nothing errored), the second
to find why the fix itself silently failed (needed a debug flag most
people wouldn't think to enable on a first pass).

### Initial, incomplete finding

Before the filtering host was installed, an existing reverse proxy
(Nginx) was already bound to all interfaces on ports 80/443 for IPv4.
When the filtering host's installer ran, its own webserver component
found IPv4 already taken on those ports and **silently fell back to
binding the IPv6 equivalent instead** — no install error, no warning,
because the install script's port configuration is written to skip a
port quietly if it's unavailable, rather than failing loudly.

At this point the install looked completely successful. It wasn't.

### The actual consequence — a live incident

An uptime monitor pinging an unrelated, already-running self-hosted app
started reporting `403 Forbidden` errors a few hours later. Root cause:
the filtering host's webserver, now bound to the IPv6 listener on ports
80/443, doesn't check the `Host` header before responding. **Any IPv6
connection landing on those ports** got served the filtering host's own
403 page — regardless of which hostname was actually being requested.

Confirmed directly: an explicit `curl` request over IPv6 to the loopback
address, with the unrelated app's hostname set in the `Host` header,
returned the filtering host's branded 403 HTML instead of the expected
app response. The unrelated app was unaffected for the large majority of
clients (anything connecting over IPv4), but silently broken for any
client or network preferring IPv6 — which is why a periodic monitor caught
it before a person did.

### First fix attempt — failed for a more subtle reason

Tried explicitly scoping the filtering host's webserver to loopback-only
addresses on both IPv4 and IPv6. The configuration parsed without error,
the process stayed running, but **the webserver subsystem silently failed
to bind to anything at all** — not even loopback. This was more confusing
than the original problem: no error in the application log, no error in
the system journal, and standard port-inspection tools showed nothing
listening on those ports for that process.

The real error only surfaced after enabling a debug-logging flag specific
to the webserver subsystem and restarting, which revealed the actual
failure buried in a log file that wasn't checked the first time: a bind
failure on the loopback IPv4 address, port already in use.

### Root cause

The original Nginx binding — "all interfaces" — is a wildcard address that,
on Linux, **inherently includes loopback**, not just external-facing
interfaces. There is no way for two separate processes to both bind the
same port when one holds the wildcard address and the other tries to bind
loopback specifically on that same port — they overlap by definition, even
though they look like different addresses. Confirmed by inspecting active
listeners directly: only a single process held each port, with no separate
explicit loopback binding from Nginx, which ruled out a deliberate Nginx
misconfiguration as the cause.

This meant the original plan — "scope the filtering host's dashboard to
loopback, keep it reachable locally" — was structurally impossible as long
as the existing reverse proxy held the wildcard address on those ports.
No amount of reconfiguring *just* the new service was going to fix it.

### Actual fix

Moved the filtering host's web interface entirely off ports 80/443 onto a
dedicated, unused port with no collision risk (checked free first via
direct port inspection, since one obvious candidate port turned out to
already be in use by another existing service). Dropped the IPv6 loopback
binding entirely — unused attack/error surface, since the only intended
client (the reverse proxy) connects over IPv4. Dropped TLS on this
specific internal hop entirely, reasoned through deliberately rather than
defaulted: a loopback-only connection between two processes on the same
machine never leaves the host, so transport encryption defends against
network interception that structurally can't happen on that hop. The real
risk for that connection type is physical theft of the device, which is
addressed by disk encryption — noted as a separate, not-yet-implemented
control, rather than papering over the gap with TLS in the wrong place.

Restarted, verified the bind with a direct port check, then verified the
actual application was responding correctly over that internal connection
before moving on to rebuild the reverse-proxy rule pointing at the new
internal port.

### What this incident actually demonstrates

The failure mode here wasn't "the install was broken" — it was an
installer designed to degrade silently meeting a network it wasn't told
about, producing a result that *looked* successful and broke something
unrelated hours later. Two separate debugging passes were needed: one to
find that a port conflict existed at all (non-obvious, since nothing
errored), and a second to find why the "fix" itself silently failed
(needed a debug flag most people wouldn't think to enable on a first
pass). Each step was confirmed with a direct, specific check rather than
assumed from absence of errors.

---

## False alarm: a privacy setting that looked like a filtering failure

### The symptom

A site that should have been blocked by an active content-filtering
policy (unrelated to the new home-lab build, already in place for over a
year via a VPN-based filtering service) loaded successfully in one
specific browser, despite the filtering service showing as active and the
new DNS-level filtering also now live. This looked, at first glance, like
either the new filtering host or the existing VPN-based filter had been
silently broken by the new setup.

### Diagnosis, in order

```
  Symptom: blocked site loads in one browser
                    │
                    ▼
  ┌───────────────────────────────────────────────┐
  │  Step 1 — test the filter directly,              │
  │  bypassing every browser entirely                  │
  │  (query the filter's own DNS stub directly)         │
  └───────────────────┬───────────────────────────┘
                       ▼
              Filter responds correctly
              → filter itself is NOT broken
                       │
                       ▼
  ┌───────────────────────────────────────────────┐
  │  Step 2 — same device, same network,               │
  │  same moment, test multiple browsers                │
  └───────────────────┬───────────────────────────┘
                       ▼
        2 of 3 browsers block it correctly
        1 browser loads the site
                       │
                       ▼
        Only variable that changed = the browser
        → problem is localized to ONE browser,
          not the network or the filtering stack
                       │
                       ▼
  ┌───────────────────────────────────────────────┐
  │  Step 3 — check that browser's own settings        │
  │  against documentation                              │
  └───────────────────┬───────────────────────────┘
                       ▼
        Found: a privacy feature in that browser
        routes some DNS through its own infrastructure,
        bypassing OS/VPN-level resolver entirely —
        unrelated to anything built in this project
```

Each step eliminates exactly one variable. The fix was found by ruling
things out in order — filter, then network, then browser config — not by
guessing based on what had just changed.

**Step 1 — test the filter directly, bypassing the browser entirely.**
Querying the blocked domain directly against the local DNS stub used by
the VPN-based filter returned the expected blocked response. This proved
the filtering policy itself was intact and functioning — the failure
wasn't in the filter.

**Step 2 — cross-browser test, same device, same moment.**
Two other browsers on the identical device, network, and VPN
configuration both correctly blocked the same domain. Only one browser
loaded it. Same device, same network, same filtering configuration,
different result by browser — this proved the discrepancy lived
specifically in that one browser, not anywhere in the filtering stack.

**Step 3 — root cause, confirmed via documentation search.**
That browser has an advanced tracking-protection feature that is
documented to route certain DNS queries through the browser vendor's own
infrastructure, **bypassing whatever resolver is configured at the
OS or VPN layer** — independent of the VPN filter, the new DNS-level
filter, the router, or any combination of them. A known, long-standing,
intentional browser behavior, entirely unrelated to anything built in
this project. The same test would have produced the identical result a
year earlier, before any of this project existed.

### Fix

Disabled that specific advanced-tracking-protection setting in the
affected browser. Re-tested: the domain now correctly fails to load,
matching the other two browsers and the direct DNS query result.

### Trade-off, stated explicitly rather than glossed over

That setting also provides general anti-fingerprinting protection
unrelated to DNS filtering. Disabling it trades away that protection in
exchange for consistent filtering behavior across browsers. Accepted as
reasonable specifically because the browsers used as daily drivers already
run their own independent tracking-protection features — the trade-off
isn't "lose all fingerprinting protection," it's "lose the redundant
overlap from one specific browser's version of it."

### Deliberately not applied everywhere

Two other devices on the network have the same theoretical exposure but
weren't touched — both are used exclusively for development and testing,
not daily browsing, so the inconsistency was accepted rather than chased
for its own sake. Filtering posture should match how a device is actually
used, not be made uniform across every device regardless of purpose.

### What this case actually demonstrates

The instinct on seeing "filtering isn't working" is to suspect whatever
just changed — in this case, the new home-lab build, which was the most
recent thing touched. The actual discipline is isolating variables instead
of assuming: testing the filter directly to rule it out first, then
testing across multiple browsers on the same device to localize *where*
the discrepancy actually lived, before forming any theory about *why*.
The root cause, once found, was completely unrelated to anything built in
this project — and the only way to know that with confidence was to rule
out the alternatives in order, rather than starting from "what did I just
change" and reverse-engineering a story from there.
