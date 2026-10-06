---
layout: welcome
title: Building a Detection Pipeline with Graylog
description: >
  From a fresh Graylog server to parsed, searchable attack data: a SIEM, an SSH
  honeypot, and the pipeline that connects them, built in parts.
caption: A SIEM, a honeypot, and the pipeline between them, built in parts.
image: /assets/img/graylog/graylog-logo.jfif
date: 2026-10-06
featured: true

# The parts, in reading order. Add part 4 here when it's published.
selected_projects:
  - _graylog-pipeline/graylog-siem.md
  - _graylog-pipeline/cowrie-honeypot.md
  - _graylog-pipeline/cowrie-graylog-pipeline.md
more_projects: /projects/
---

This project builds a working detection pipeline in my homelab, one piece at a time.
It starts with a central SIEM, adds a honeypot to generate real attack data, then
connects the two so every login attempt arrives parsed and searchable. See the
[Lab Build](/lab-build/) page for the network it runs on.

## The build

<!--projects-->
