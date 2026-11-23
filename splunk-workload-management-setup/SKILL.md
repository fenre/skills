---
name: splunk-workload-management-setup
description: "Configure Splunk Workload Management: pools, workload and admission rules, cgroups prerequisites, guardrails for long or expensive searches. Use to reserve resources or limit costly searches."
---

# Splunk Workload Management Setup

This skill renders Splunk Enterprise Workload Management configuration for
Linux-based Splunk Enterprise deployments. It is not a Splunk Cloud workflow.

## Agent Behavior

Never ask for secrets in chat. This workflow manages configuration files and
local Splunk CLI commands; it does not need credentials in normal local use.

Before enabling workload management, confirm the target host meets the Linux
cgroups requirements and review all predicates.

## Quick Start

Render a balanced policy:

```bash
bash skills/splunk-workload-management-setup/scripts/setup.sh \
  --profile balanced \
  --critical-role admin
```

Render and enable workload management plus admission rules:

```bash
bash skills/splunk-workload-management-setup/scripts/setup.sh \
  --phase apply \
  --profile ingest-protect \
  --enable-workload-management \
  --enable-admission-rules
```

## What It Renders

- `workload_pools.conf` with search, ingest, and misc categories
- `workload_rules.conf` with placement, monitoring, and optional admission rules
- `workload_policy.conf` with admission control enablement
- helper scripts for preflight, apply, and status

Admission rules are rendered as `[search_filter_rule:<name>]` stanzas in
`workload_rules.conf`, matching Splunk's documented storage model.

Read `reference.md` before applying in distributed deployments.
