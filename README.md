# Home Lab DNS Filtering — Pi-hole Architecture & Investigation Log

**Status:** Complete — operational, verified end-to-end
**Stack:** Raspberry Pi 4B, Debian, Pi-hole, Nginx, systemd, Let's Encrypt (DNS-01)

This repo documents the design, build, and debugging process behind a network-wide
DNS filtering layer for a home lab. There's no application code here — the
artifact is the architecture and the reasoning, not a thing to clone and run.
That's deliberate: most of what differentiates a junior infrastructure engineer
from someone following a tutorial isn't *that* something works, it's whether
they can explain why it's built the way it's built, and what they ruled out
along the way.

## Why this exists

Pi-hole is a well-documented, widely-deployed tool. Standing one up isn't
novel. What's documented here is the process around it:

- A security review methodology applied to an unfamiliar install script,
  rather than trusting a `curl | bash` one-liner on faith
- A failure-domain analysis of what "single point of failure" actually means
  for home network DNS, and why the obvious fix (a second Pi-hole) doesn't
  solve it
- A live incident — caused by a port-binding conflict — diagnosed from a
  misleading symptom down to root cause across multiple debugging passes
- A deliberate decision to **not** fix a network segmentation gap, because
  the fix would have weakened a security boundary that mattered more than
  the convenience it would have bought

## Read this in order

```
  README.md  ──────────────────────────────────────────────
  (you are here)        Overview, why this exists
        │
        ▼
  ARCHITECTURE.md       What was built
  ─────────────────     ───────────────────────────────────
                         • Resolver vs. filter
                         • Failure-domain reasoning
                         • Final stack
        │
        ▼
  SECURITY-REVIEW.md    How the install was vetted
  ─────────────────     ───────────────────────────────────
                         • 4-category script review
                         • Threat model
                         • Blast-radius analysis
        │
        ▼
  INVESTIGATIONS.md     What went wrong, and what didn't
  ─────────────────     ───────────────────────────────────
                         • Real incident: port conflict
                         • False alarm: ruled out correctly
```

Each file stands alone — read just `ARCHITECTURE.md` for the design, or
jump straight to `INVESTIGATIONS.md` for the debugging process. The order
above is just the narrative sequence: design it, vet it, build it, debug
it.

1. **[ARCHITECTURE.md](./ARCHITECTURE.md)** — the network design, the
   resolver-vs-filter distinction, failure-domain reasoning, and the final
   stack
2. **[SECURITY-REVIEW.md](./SECURITY-REVIEW.md)** — the install-script
   review process, threat model, and blast-radius analysis
3. **[INVESTIGATIONS.md](./INVESTIGATIONS.md)** — two debugging case
   studies: a real incident caused by a port conflict, and a false alarm
   that looked like a filtering failure but wasn't

## Scope note

Network details below (IP ranges, hostnames) are illustrative, not the real
addressing scheme. The reasoning, sequencing, and technical decisions are
accurate; the specific values are not, by design.
