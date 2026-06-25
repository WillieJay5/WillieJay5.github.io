---
layout: page
title: Threat Model
description: >
  What the lab protects, from whom, and the blast radius when a host is
  compromised — the design basis for every firewall rule, detection, and
  incident-response exercise that follows.
menu: true
order: 2
---

> This document is the foundation for the rest of the site. Firewall and
> segmentation writeups *implement* the zone model defined here; detection
> engineering writeups *monitor* the boundaries defined here; incident-response
> writeups *test whether the blast-radius assumptions hold*. Where a later page
> makes a design choice, it should be able to point back to a line in this
> document and say "because of this."
{:.note}

## Overview

Most homelab writeups answer the question "what did I install?" This one answers
a different question first: **what am I actually protecting, who or what am I
protecting it from, and what happens when — not if — something inside the lab is
compromised?**

The lab is deliberately hostile by design. It runs adversary-simulation tooling
([Atomic Red Team](https://atomicredteam.io/)), it will host intentionally
weak and out-of-date endpoints so they can be attacked and detected, and it
exists specifically to generate the kind of telemetry a defender sees during a
real intrusion. That premise drives the single most important decision in the
whole architecture:

**Compromise of a lab workload is treated as an expected operating condition,
not an incident.** The architecture's job is not to prevent a lab host from being
owned — that is sometimes the *goal* of an exercise. Its job is to guarantee that
a compromise stays inside the lab and never reaches the home network, my personal
identity, or the hypervisor that hosts everything.

Everything below follows from that assume-breach posture.

## Scope and methodology

The model uses three lenses, applied in order:

1. **Asset valuation** — rank what is being protected by the real-world harm of
   its loss, *not* by where it sits on the network.
2. **Trust zones and boundaries** — partition the environment into zones with
   explicit trust levels, then define which flows between them are permitted.
   This is a trust-boundary / data-flow view; per-component threat enumeration
   borrows from STRIDE where it adds signal (notably spoofing and elevation
   across the hypervisor boundary).
3. **Blast-radius analysis** — for each high-value zone, assume an adversary has
   already achieved code execution somewhere lower-trust and trace exactly how
   far they can reach. This is where the assume-breach posture earns its keep.

**In scope:** network and hypervisor-level containment of compromise originating
inside the lab; the single exposed ingress (VPN); egress control; the integrity
of the defensive tooling (SIEM, firewall, hypervisor).

**Explicitly out of scope** (stated honestly rather than pretended away):
physical intrusion into the home, supply-chain compromise of upstream OS/package
repositories, and a determined, well-resourced targeted adversary. The lab is a
learning environment; modeling a nation-state against it would be theatre.

## Assets — what I'm protecting

The defining feature of this model is that **the highest-value asset sits
*outside* the lab and the lab never legitimately communicates with it.** A normal
corporate threat model protects the assets *inside* the perimeter; here the
crown jewel is the home network and personal identity, and the lab is the
untrusted thing it must be protected *from*.

| Priority | Asset | Why it ranks here | Trust the lab is granted to it |
| --- | --- | --- | --- |
| 1 | **Home network + personal identity** (personal devices, accounts, anything tying the lab to a real person or address) | Its compromise is the only outcome with real-world, irreversible harm. | **None.** No lab zone may reach it. Ever. |
| 2 | **Hypervisor control plane** (Proxmox `pve01`) | Owning it owns every VM, every snapshot, and all stored telemetry — catastrophic, lab-wide blast radius. | Management only, from the admin path. |
| 3 | **SIEM integrity + availability** (Graylog on `siem01`) | The defender's eyes. An attacker who can blind or poison it operates unseen; it also has visibility *into* every zone, making it a prized pivot target. | Receives logs (push, one-way). |
| 4 | **Firewall / routing control plane** (OPNSense) | Enforces every boundary in this document. Its compromise dissolves all segmentation at once. | Management only, from the admin path. |
| 5 | **Lab workloads** (Windows endpoints, Atomic Red Team targets, future intentionally-weak hosts) | Deliberately sacrificial. Their compromise is often the *point*. | Low / assumed-hostile. |

The ordering is the whole insight: **the things I most need to protect are the
things the lab is never allowed to touch, and the things most likely to be
compromised are the things I value least.** Segmentation exists to keep those two
facts from ever meeting.

## Adversaries and threat scenarios

| Adversary | Realistic? | What they're trying to do |
| --- | --- | --- |
| **Deliberately-introduced attacker** — Atomic Red Team executions, intentionally weak VMs, simulation tooling | By design, continuously | Execute, persist, move laterally, reach C2, exfiltrate — so I can detect each step |
| **Opportunistic external attacker** | Yes — any internet-exposed listener gets scanned within minutes | Find and exploit the one exposed surface (the VPN endpoint / OPNSense WAN), then establish a foothold |
| **Compromised admin/VPN client** (my own laptop) | Yes, and routinely under-modeled | Use already-authenticated VPN access to pivot from the lab into the home network or the management plane |
| **A "popped" lab host pivoting** | Yes — this is the expected condition | Reach the home network, escape to the hypervisor, reach/poison the SIEM, or call out to the internet |

The third row matters more than people give it credit for. A VPN tunnel is
commonly treated as a *trusted* pipe; here it is treated as a **boundary that can
itself be the source of compromise.** WireGuard terminating on OPNSense, scoped
by firewall rules to reach *only* the lab, means that even a fully-compromised
admin laptop cannot use its VPN session to reach the home network.

## Trust zones and boundaries

The lab is partitioned into trust zones. Each zone has a trust level and an
explicit, default-deny set of permitted flows. Hosts are referred to by generic
role names only (`siem01`, `win-ep01`, …); real names are never published.

```text
                        ┌──────────────────────────────┐
   Internet (untrusted) │  ── only inbound: WireGuard ──▶│  OPNSense (WAN edge)
                        └──────────────────────────────┘        │
                                                                 │ terminates VPN
              ╔══════════════════════════ TRUST BOUNDARY ════════╪══════════════╗
              ║                                                   ▼              ║
              ║   Z-MGMT (highest)            Z-SVC (high)        VPN client      ║
              ║   ┌───────────────┐           ┌───────────────┐  scoped to lab   ║
              ║   │ pve01  (Proxmox│           │ siem01 (Graylog│  only           ║
              ║   │        mgmt)   │           │ ngx01  (HTTPS) │                 ║
              ║   │ opnsense (mgmt)│           └───────▲───────┘                 ║
              ║   └───────────────┘                   │ logs (push, one-way)     ║
              ║                                        │                          ║
              ║   Z-WORK (low / assumed hostile)       │                          ║
              ║   ┌───────────────────────────────────┴───────┐                  ║
              ║   │ win-ep01, atomic targets, future weak hosts│                  ║
              ║   └────────────────────────────────────────────┘                 ║
              ╚═══════════════════════════════════════════════════════════════════╝
                          ▲
                          │  NO PATH EXISTS across this line ↓
              ┌───────────┴──────────────────────────────────────────────────────┐
              │ Z-HOME (protected, never reachable by any lab zone)               │
              │ personal devices, accounts, identity                              │
              └───────────────────────────────────────────────────────────────────┘
```

> A polished version of this diagram will replace the ASCII sketch above.
> `[PLACEHOLDER — render trust-boundary diagram → /assets/img/lab/trust-zones.png]`
{:.note}

| Zone | Trust | Contains | May initiate connections to |
| --- | --- | --- | --- |
| **Z-HOME** | Protected (not "trusted by the lab" — *off-limits to it*) | Personal devices, identity | n/a — outside the lab |
| **Z-MGMT** | Highest | Proxmox + OPNSense management interfaces | Nothing outbound to the internet; reachable only via the admin path |
| **Z-SVC** | High | `siem01` (Graylog), `ngx01` (internal HTTPS) | Nothing outbound to the internet; receives logs, does not originate connections into Z-WORK |
| **Z-WORK** | Low / assumed hostile | Windows endpoints, Atomic targets, future weak hosts | **Default-deny everywhere**, except a one-way log/agent path to `siem01` |
| **VPN** | Conditional | Authenticated admin session | Lab zones only — scoped by OPNSense rules, never Z-HOME |

The current build collapses some of this — `siem01` and `ngx01` live together on
the Debian host `prometheus`, and Z-WORK is still being populated as Windows VMs
come online. The zone model above is the **target state**; segmentation writeups
will document the migration from the current flatter layout toward it, and that
migration is itself part of the portfolio.

### Documentation addressing convention

Because the site is public and git history is permanent, no real addressing
appears anywhere. All examples across the site use this convention; the numbers
below are **illustrative placeholders, not the deployed scheme**:

- "Internet-facing" / WAN examples use [RFC 5737](https://datatracker.ietf.org/doc/html/rfc5737)
  documentation ranges (`198.51.100.0/24`, `203.0.113.0/24`) so they are
  unmistakably fake.
- Internal zones are written as `10.10.<zone>.0/24` placeholders — e.g.
  `Z-MGMT = 10.10.10.0/24`, `Z-SVC = 10.10.20.0/24`, `Z-WORK = 10.10.30.0/24`.
  These are documentation values only and do not reflect real subnets.

Treating sanitization as a deliberate, documented choice — rather than something
done quietly after the fact — is itself a competence signal worth stating out loud.

## Attack surfaces

Ordered by how much an attacker would want them:

1. **The VPN listener (the only inbound path).** A single WireGuard endpoint on
   OPNSense. WireGuard's design helps here: it does not respond to unauthenticated
   packets, so the listener is effectively invisible to scanners without a valid
   key. This is the lone deliberately-exposed surface.
2. **Egress from Z-WORK to the internet.** The path a compromised workload would
   use for C2 and exfiltration. Closing it is more important than most inbound
   controls — see below.
3. **The hypervisor boundary (VM → host).** A VM escape is the catastrophic case;
   it is the reason Z-MGMT is the most protected zone.
4. **The SIEM ingest path.** Logs flow *from* hostile hosts *into* `siem01`.
   Maliciously-crafted log content (log injection aimed at parsers/extractors) is
   a real concern for a SIEM, so the ingest path is push-only and `siem01` never
   initiates connections back into Z-WORK.
5. **The admin/VPN client itself.** Modeled as potentially compromised; scoping
   keeps a popped client from reaching Z-HOME or Z-MGMT.

## Blast-radius analysis

The core of the model. For each scenario, assume the adversary already has code
execution and trace how far they get.

**Scenario A — a Z-WORK workload is popped (the expected case).**
An attacker owns `win-ep01`. Reachability: its own segment, and a one-way log
path to `siem01`. *Not* reachable: Z-HOME (no route exists), the internet
(default-deny egress kills C2 and exfil), Z-MGMT (management plane is isolated),
and `siem01` as a pivot (it accepts logs but does not expose lateral services to
Z-WORK). The attacker is loud and boxed in — which is exactly the telemetry the
lab is built to capture. *This is the success case, not the failure case.*

**Scenario B — the admin/VPN client is popped.**
An attacker rides an authenticated VPN session. Reachability: the lab zones the
session is scoped to. *Not* reachable: Z-HOME, because the WireGuard rules
contain the tunnel to the lab. Reaching Z-MGMT still requires separate
management credentials, so a popped client is not automatically a popped
hypervisor.

**Scenario C — `siem01` is popped (high impact).**
The SIEM has visibility everywhere, so this hurts: the attacker can read
telemetry and attempt to blind detection. Containment relies on `siem01` having
no internet egress, the log path being push-only (so a popped workload could not
easily have reached it in the first place), and Z-SVC being segmented from
Z-WORK. The honest residual risk: an attacker on `siem01` sees a lot. This ranks
it asset #3 and justifies hardening it like production.

**Scenario D — the hypervisor is popped (catastrophic).**
Owning `pve01` is game-over *for the lab* — every VM and snapshot is exposed. The
one boundary that must still hold is the line to Z-HOME, which remains a physical/
routing separation rather than something a hypervisor compromise dissolves. This
scenario is why Z-MGMT is the highest-trust zone, why VM-escape is the top
modeled hypervisor threat, and why hypervisor hardening gets its own writeup.

## Controls and rationale

Each control maps to a threat it answers. Firewall writeups will implement these
as concrete rulesets; the rationale lives here so the rules never have to
re-justify themselves.

| Control | Threat answered | Residual risk |
| --- | --- | --- |
| **VPN-only ingress** (single WireGuard endpoint, no other inbound) | Opportunistic external attacker; reduces exposed surface to ~one silent listener | Compromise of a valid key / the client (covered by scoping) |
| **Default-deny inbound** at every zone boundary | Lateral movement from a popped host | Misconfigured allow rule (caught by monitoring) |
| **Default-deny *egress* from Z-WORK** | C2 and exfiltration from a popped workload | DNS / allowed-protocol tunneling (a future detection target) |
| **Inter-zone deny** (Z-WORK ↔ Z-SVC/Z-MGMT) | Pivot from sacrificial host to defensive infrastructure | Trust in the deny rules themselves (tested by IR exercises) |
| **Management-plane isolation** (Z-MGMT reachable only via admin path) | Privilege escalation toward the hypervisor | Compromise of admin credentials |
| **One-way, push-based log shipping** to `siem01` | SIEM used as a pivot; log-injection abuse | Crafted log content targeting parsers (hardening + parser care) |
| **VPN scoped to lab only** | Popped admin client pivoting into Z-HOME | n/a for Z-HOME; client still trusted within the lab |
| **Hypervisor hardening** | VM escape to `pve01` | Unknown hypervisor 0-day (accepted, out-of-scope class) |

The pattern worth noticing: **egress control does more containment work than
inbound control.** Once a lab host is assumed compromised, what matters is what it
can *reach* — so default-deny outbound, not just inbound, is the load-bearing
decision.

## How this drives the rest of the portfolio

This is why the threat model comes first: it generates the work.

- **Firewall & segmentation writeups** implement the zone matrix above. Each rule
  cites the boundary and threat it enforces, so the post reads as "here is the
  control for threat X," not "here are my rules."
- **The SIEM's first high-value detections come straight out of this document.**
  Every flow the model says must *never* happen — a Z-WORK host reaching Z-HOME,
  any egress from Z-WORK to the internet, lateral movement toward `siem01` or
  Z-MGMT — becomes an alert. These boundary-violation detections are the ideal
  first detections precisely because they fire on *architecturally forbidden*
  events, giving them a near-zero false-positive rate. They map cleanly to
  ATT&CK Lateral Movement (TA0008), Command and Control (TA0011), and
  Exfiltration (TA0010), which the detection-engineering and IR writeups will tag
  technique-by-technique.
- **Incident-response exercises test whether the blast-radius assumptions hold.**
  Each Atomic Red Team scenario is, in effect, an attempt to violate one of the
  containment claims in the blast-radius section. If a "should-be-impossible"
  flow succeeds, that is a finding — and the writeup of closing it is some of the
  most honest, highest-signal content the portfolio can carry.

In short: the deny rules become firewall posts, the impossible flows become
detections, and the IR exercises check that both are telling the truth.
