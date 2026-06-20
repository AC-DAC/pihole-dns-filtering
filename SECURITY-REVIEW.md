# Security Review Process

## Why review before running

Pi-hole's official install path is a `curl | bash` one-liner. Piping a
remote script straight into a shell means trusting unreviewed, unsigned
content at the exact moment it executes — no maintainer signature, no
packaging review pipeline, nothing standing between "this is what the
script claims to do" and "this runs with whatever privileges it's granted,"
even over a fully valid HTTPS connection.

Compare that to a package manager: packages are signed and verified
automatically before anything executes, with a review pipeline behind the
package in the first place. `curl | bash` has neither.

Decision made: download the script first, review it locally, then execute
deliberately — rather than piping it directly into a shell.

```
  Downloaded script
         │
         ▼
  ┌─────────────────────────────────────────────────┐
  │  1. Unexplained external fetches                 │
  │     grep for curl/wget to unknown domains         │
  └─────────────────────┬─────────────────────────────┘
                         ▼ clean
  ┌─────────────────────────────────────────────────┐
  │  2. Obfuscation                                   │
  │     grep for base64 blobs, eval of external data  │
  └─────────────────────┬─────────────────────────────┘
                         ▼ clean
  ┌─────────────────────────────────────────────────┐
  │  3. Excessive permissions                         │
  │     grep for chmod 777, setuid/setgid, sudoers    │
  └─────────────────────┬─────────────────────────────┘
                         ▼ clean
  ┌─────────────────────────────────────────────────┐
  │  4. Rogue persistence                             │
  │     grep for crontab, systemctl enable, rc edits  │
  └─────────────────────┬─────────────────────────────┘
                         ▼ empty result —
                           ambiguous, not automatically "safe"
                         │
                         ▼
              Confirm what empty actually means:
        nothing persists  vs.  persistence handled elsewhere
                         │
                         ▼
              Deferred to package's own install hooks
              (more trustworthy mechanism, not a gap)
                         │
                         ▼
                  Proceed with execution
```

A script only needs to fail *one* category to stop the process. Reaching
the bottom means it passed all four — not that the last category was
skipped.

## Review framework — four categories

Rather than "read the whole thing and look for anything that seems bad,"
the review used four concrete categories, each with a specific thing to
grep for:

**1. Unexplained external fetches**
Nested `curl`/`wget` calls to domains other than the expected source,
especially anything that gets piped into a second `bash`/`sh` invocation.
A legitimate installer fetching its own components from its own
infrastructure is normal; an installer quietly reaching out to an unrelated
third-party domain is not.

**2. Obfuscation**
Base64-encoded blobs, or `eval` applied to anything sourced from outside
the script. Note the distinction: `eval` of a variable the script defines
internally is completely normal shell practice and not a red flag on its
own — the concern is specifically `eval` of externally-fetched content,
where you can't read what's about to execute.

**3. Excessive permissions**
`chmod 777`, setuid/setgid bits being set, or unexplained writes to
`sudoers`. Installers legitimately need elevated permissions for specific,
narrow operations (installing a system service, writing to `/etc`) — the
review is for permission grants that are broader than the stated purpose.

**4. Rogue persistence**
Anything that installs a `crontab` entry, enables a systemd service, or
edits shell rc files *beyond* what the tool is supposed to do. The
interesting finding in this specific review was an **empty grep result**
for this category — which could mean two very different things: either
nothing persists, or persistence is handled somewhere else the script
doesn't show. Confirming which one matters; assuming silence means safety
is its own mistake.

## Resolving the empty-grep ambiguity

For category 4, the empty result was correctly interpreted as "this
particular install script doesn't handle service-enablement itself" — the
actual persistence (the equivalent of `systemctl enable`) was deferred to
the underlying package's own post-install hooks, which is the more
trustworthy mechanism of the two, not a gap in the review. The distinction
matters: "I didn't find it because it isn't there" and "I didn't find it
because I was looking in the wrong place" produce the same grep output and
require different follow-up.

**Result of the review:** clean across all four categories. Proceeded with
execution.

## Threat model — why this specific service matters more than generic compromise

Most "what if this gets hacked" discussions about a home server default to
generic outcomes — crypto-mining, the box getting recruited into a botnet.
Those risks are real but **generic to any compromised Linux machine** —
nothing about this specific service makes it more or less likely to be
targeted for those purposes than any other always-on device on the
network.

What makes a DNS filtering host specifically higher-value as a target is
its **position in the request path**, not its function. Sitting as the
network's primary DNS resolver means it sees every domain every device
tries to reach, *before* any of those connections happen — and it actively
**answers** those queries. A compromised instance doesn't just passively
observe traffic; it could return a wrong, attacker-controlled IP address
for a real, legitimate domain. That's active manipulation of where traffic
goes, not passive logging of where it was going. Materially worse outcome
than either generic compromise category.

## Bounding the blast radius

If the filtering host itself were compromised, what does that actually
grant an attacker? Working through it deliberately rather than assuming
"everything":

```
                    Filtering host compromised
                              │
                              ▼
              ┌───────────────────────────────┐
              │   Direct, automatic result:     │
              │   DNS-level effects only         │
              │   (wrong IPs returned for         │
              │    real domains)                  │
              └───────────────────────────────┘
                              │
          requires an ADDITIONAL, independent failure
          to reach anything beyond DNS  ──┐
                              │            │
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                 ▼
    ┌───────────────┐ ┌───────────────┐ ┌────────────────┐
    │  Banking creds  │ │  Other network   │ │  Other LAN       │
    │  ──────────────  │ │  storage          │ │  systems          │
    │  blocked by TLS  │ │  ────────────     │ │  ────────────     │
    │  cert validation │ │  blocked by        │ │  blocked by        │
    │  (separate layer)│ │  separate auth     │ │  separate auth     │
    └───────────────┘ └───────────────┘ └────────────────┘
            NOT reached automatically — each needs its own,
                    independent compromise to chain

    ┌─────────────────────────────────────────────────────┐
    │  One exception — fully reachable, no chaining needed:   │
    │  host recruited into a botnet → ISP flags/throttles      │
    │  the connection. Accepted as residual risk, same as       │
    │  for any internet-connected device.                        │
    └─────────────────────────────────────────────────────┘
```

- **DNS-level effects only**, by default. The compromise doesn't, on its
  own, hand over banking credentials (TLS certificate validation is an
  independent defense layer a DNS-level attacker can't forge), grant access
  to other storage on the network (separate authentication), or control
  other systems on the LAN. Each of those would require additional,
  independent failures chained on top of the DNS compromise — not
  automatic consequences of it.
- **The one fully legitimate, undiminished risk:** the host being
  recruited into a botnet, with the household's connection getting
  flagged or throttled by the ISP as a result. That risk isn't mitigated
  by anything specific to this build and is treated as accepted residual
  risk, same as it would be for any internet-connected device.

Being explicit about what a compromise *doesn't* grant is as important as
naming what it does — overstating the blast radius leads to either wasted
effort hardening things that were never actually at risk, or to dismissing
the analysis entirely because it sounds alarmist.

## Recovery sequencing

If something looked compromised or behaving unexpectedly, the response
order matters:

1. **Fast containment first** — removing the host from the DNS path at the
   router (a two-click, seconds-scale change, or simply confirming the
   secondary DNS fallback is already covering for it) takes the
   compromised device out of the live request path immediately.
2. **Full rebuild second** — reimaging the host is the correct *eventual*
   remediation, but it's slow, and doing it first leaves the compromised
   device live and answering queries for however long the reimage takes.
   Containment buys time; it isn't itself the fix, but it has to happen
   before the fix, not after.

The general principle: the fast, reversible action that limits exposure
comes before the slow, thorough action that actually resolves the root
cause — not because the thorough fix doesn't matter, but because exposure
time compounds while you're doing it.
