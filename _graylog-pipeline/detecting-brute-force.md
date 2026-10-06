---
layout: project
title: "Detecting a Brute-Force Attack"
description: >
  The payoff: an Event Definition that fires on SSH brute-force, a dashboard that
  shows the attack in real time, and a live test with Hydra from Kali.
image: /assets/img/graylog/graylog-logo.jfif
date: 2026-10-07
caption: Turning parsed logs into a brute-force alert, a dashboard, and a live test.
links:
  - title: Project overview
    url: /projects/graylog-pipeline/
  - title: Lab Build
    url: /lab-build/
---

We have logs flowing in and parsed into clean fields. Now let's actually *use* them.
{:.lead}

Everything up to this point was plumbing: a SIEM to collect logs, a honeypot to generate them, and a pipeline to break them into searchable fields. That work only pays off when the data tells us something. In this final part we build the thing a blue team actually cares about — a detection that fires on an attack — then a dashboard to see it happen, and finally we prove it works by attacking the honeypot ourselves and watching the alert trip.

By the end, a single source IP hammering the honeypot with failed logins will light up a dashboard and raise an event in Graylog, all on its own.

* toc
{:toc .large-only}

## Part 1: Build the brute-force detection rule

This is the centerpiece: an **Event Definition** that fires when a single source IP racks up too many failed logins in a short window — the signature of an SSH brute-force.

Go to **Alerts → Event Definitions → Create Event Definition**. It's a multi-step wizard.

**Event Details.** Title it `Cowrie SSH Brute Force`. Click Next.

**Condition.** This is the step that matters. At the top, set the **Condition Type** to **Filter & Aggregation**, not plain "Filter". The grouping and count options only appear once aggregation is selected — if all you see is a search query field, you're on the plain Filter view and need to switch the type. Then fill in:

- **Search Query:** `eventid:cowrie.login.failed`
- **Streams:** `Cowrie Honeypot`
- **Search within:** 5 minutes
- **Execute every:** 1 minute
- **Group by Field(s):** `src_ip`
- **Create Events if:** `count() > 20`

**Fields.** This custom-fields / lookup-table step is optional enrichment. Leave it empty for the core detection.

**Notifications.** Also optional. The event still fires and shows up under **Alerts** without one — you can wire up email or another channel later.

**Summary.** Confirm and save. Make sure it's enabled and shows in the Event Definitions list.

### Why group by `src_ip`

Without grouping, the rule counts *all* failed logins together, so ten different IPs failing twice each would falsely trip a "20 failures" threshold. Grouping by `src_ip` means the rule fires only when a *single* IP crosses the threshold — which is what actually indicates a brute-force coming from that source.

### Tuning the threshold

More than 20 in 5 minutes is a reasonable starting point. Lower it if your test attempts don't reach it; raise it to cut noise in a real deployment. Remember the rule runs on its execution interval — every minute — so the alert can lag the attack by up to a minute. That delay is normal.
{:.note}

## Part 2: Build the dashboard

The dashboard is the visual, screenshot-able deliverable. Go to **Dashboards → Create new dashboard**. Set the dashboard's stream to **Cowrie Honeypot** and the time range to something generous — Last 1–4 hours — while you build. Add widgets with the **Add → Aggregation** control. Each widget is defined by what you group by (Rows), what you measure (Metric, usually `count()`), and the visualization.

Five widgets tell the story:

1. **Failed Logins Over Time** — Metric `count()`, group by `timestamp` (Graylog buckets time automatically), **Line chart**. Query `eventid:cowrie.login.failed`. This is the headline "attack in progress" visual.
2. **Top Usernames** — Metric `count()`, group by `username`, top 10, **Bar chart** or **Data Table**. Query `eventid:cowrie.login.failed`.
3. **Top Passwords** — Metric `count()`, group by `password`, top 10, **Bar chart** or **Data Table**. Query `eventid:cowrie.login.failed`.
4. **Top Source IPs** — Metric `count()`, group by `src_ip`, top 10, **Bar chart** or **Data Table**. Query `eventid:cowrie.login.failed`. During an attack the attacker's IP dominates this — that's the brute-force signature in one picture.
5. **Login Outcomes** — either two single-number widgets (`eventid:cowrie.login.failed` and `eventid:cowrie.login.success`), or one Aggregation grouped by `eventid` shown as a pie or table. This shows the failed-versus-successful ratio.

<!-- Drop a dashboard screenshot here once it's populated:
![Cowrie brute-force dashboard](/assets/img/graylog/cowrie-dashboard.png) -->

A few things that save headaches:

- Leave the dashboard's **global query empty** and filter per widget. The Login Outcomes widget needs to see both failed *and* successful events, so it can't inherit a "failed only" global filter.
- Field names autocomplete from real data as you type. If `eventid`, `username`, `password`, or `src_ip` autocomplete differently, use whatever the autocomplete shows — that's the ground truth of how your events actually parsed.
- Suggested layout: "Failed Logins Over Time" wide across the top, the three "Top" widgets in a row beneath it, and "Login Outcomes" tucked in a corner. Save often.
{:.note}

## Part 3: Test it with Hydra from Kali

Time to tie it all together: brute-force the honeypot from a separate machine and watch the detection fire and the dashboard fill in. Use a *different* machine so the source IP differs from the honeypot itself.

### Get Kali onto the same network

On VMware Workstation, the default NAT mode puts the VM on a private VMware network that can't reach your Proxmox VMs. Switch the adapter to **Bridged** (VM → Settings → Network Adapter → Bridged), which puts Kali directly on the LAN with a `192.168.100.x` address. Renew its lease or reboot:

```
sudo dhclient -r && sudo dhclient
```

Then confirm you can actually reach the honeypot:

```
ip addr show | grep "inet " | grep -v 127.0.0.1
ping -c 3 192.168.100.54
nc -zv 192.168.100.54 2222
```

You want a `192.168.100.x` address, successful pings, and port 2222 open.

### Prepare the wordlists

For a controlled demo, plant your one valid password at the end of a small list, and include the valid username in a userlist:

```
printf '123456\npassword\nletmein\nqwerty\nadmin\nroot\ntoor\nchangeme\nwelcome\nmonkey\ndragon\nabc123\niloveyou\nsunshine\nprincess\nfootball\ncharlie\nsuperman\nbatman\ntrustno1\n' > /tmp/wordlist.txt
echo 'YOUR_REAL_PASSWORD' >> /tmp/wordlist.txt

printf 'root\nadmin\nadministrator\nuser\ntest\noracle\npostgres\nmysql\nubuntu\nguest\n' > /tmp/users.txt
echo 'YOUR_VALID_USERNAME' >> /tmp/users.txt
```

Make sure the valid username and password — as set in Cowrie's `userdb` — are actually in these lists, or the success event will never register. For a more realistic run, use `/usr/share/wordlists/rockyou.txt` (unzip it first with `sudo gunzip` if needed) and Ctrl+C after 30–60 seconds.

### Run the attack

```
hydra -L /tmp/users.txt -P /tmp/wordlist.txt -s 2222 -t 4 -f 192.168.100.54 ssh
```

Breaking that down: `-L` user list (capital L), `-P` password list, `-s 2222` port, `-t 4` four parallel tasks (modest is fine — you want steady volume, not speed), `-f` stop on the first valid pair, then the target `192.168.100.54` and protocol `ssh`.

Watch the multiplication: usernames × passwords. Keep the lists small, or just Ctrl+C once you've generated enough traffic. You only need to cross the threshold — more than 20 failures in 5 minutes.

### Watch it fire

In Graylog, open the dashboard, set the range to **Last 5 minutes**, and turn on auto-refresh (5–10 seconds). As Hydra runs:

- **Failed Logins Over Time** spikes.
- **Top Source IPs** is dominated by Kali's IP.
- **Top Usernames** and **Top Passwords** fill in with what Hydra is trying.

Then check **Alerts → Events** — the `Cowrie SSH Brute Force` event fires once Kali's IP crosses the threshold. Give it up to a minute for the rule's interval.

> **A note on scope.** This is your own honeypot, on your own network, built to be attacked. Running Hydra against `192.168.100.54:2222` here is a legitimate test of your own detection. The same tool pointed at any system you don't own is a different matter entirely.
{:.note}

## Wrapping up

That's the whole pipeline, end to end. A raw login attempt leaves the honeypot, ships over GELF, lands in its own stream and index, gets parsed into fields, and now — the part that makes it security work rather than log collection — trips a detection and shows up on a dashboard the moment an attacker crosses the line.

From here there's plenty to extend: wire the event to an email or chat notification, enrich `src_ip` with GeoIP to map where attempts come from, or map the behavior to MITRE ATT&CK and start treating each alert like a real investigation. But the core loop — collect, parse, detect, visualize — is done and working.
