---
name: splunk-alerts
description: "Splunk Alerts and Saved Searches reference for savedsearches.conf configuration. Use when: (1) Creating scheduled or real-time alerts in savedsearches.conf, (2) Configuring alert trigger conditions (counttype, relation, quantity), (3) Setting up alert actions (email, webhook, log event, CSV output), (4) Configuring alert throttling and suppression, (5) Writing cron schedule expressions for alert scheduling, (6) Troubleshooting savedsearches.conf startup errors such as 'Invalid key' or 'Cannot parse into key-value pair', (7) Packaging apps with alerts for AppInspect and Splunk Cloud vetting."
---

# Splunk Alerts & Saved Searches Reference

## Overview

Alerts in Splunk use saved searches to monitor for specific events. A saved search runs on a schedule (or in real time), evaluates trigger conditions against the results, and executes alert actions when conditions are met. All alert configuration is stored in `savedsearches.conf`.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ALERT ARCHITECTURE                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  savedsearches.conf          TRIGGER ENGINE           ALERT ACTIONS          │
│  ────────────────            ──────────────           ─────────────          │
│                                                                              │
│  ┌─────────────┐          ┌──────────────────┐     ┌──────────────────┐    │
│  │ [Alert Name]│          │ Trigger          │     │ action.email     │    │
│  │ search = ...│ ───────▶ │ Evaluation       │ ──▶ │ action.webhook   │    │
│  │ cron = ...  │  results │ counttype        │     │ action.logevent  │    │
│  │ enableSched │          │ relation         │     │ action.lookup    │    │
│  │ counttype   │          │ quantity         │     │ alert.track      │    │
│  │ relation    │          │                  │     │ action.script    │    │
│  │ quantity    │          │ Custom condition │     │ (custom actions) │    │
│  └─────────────┘          └──────────────────┘     └──────────────────┘    │
│                                    │                                         │
│                            ┌───────┴────────┐                               │
│                            │  Throttle /    │                               │
│                            │  Suppress      │                               │
│                            │  alert.suppress│                               │
│                            └────────────────┘                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Alert Types

| Type | Description | Use When |
|------|-------------|----------|
| **Scheduled** | Runs on a cron schedule, evaluates results against trigger conditions | Most alerting use cases — periodic monitoring |
| **Real-time** | Runs continuously, triggers on every result or within a rolling time window | Critical events requiring immediate notification |

Scheduled alerts are recommended for most use cases. Real-time alerts consume significantly more resources.

---

## savedsearches.conf — Complete Key Reference

### CRITICAL: Valid Key Names

The following table lists EVERY valid key for alert configuration. Keys not in this list will cause `"Invalid key"` errors on Splunk startup.

#### Scheduling & Execution Keys

| Key | Type | Description | Example Values |
|-----|------|-------------|----------------|
| `search` | string | The SPL query. Multi-line requires `\` continuation. | `index=myindex \| stats count` |
| `enableSched` | bool | Enable scheduled execution. **NOT** `is_scheduled`. | `1` or `0` |
| `cron_schedule` | string | Cron expression (5 fields: min hour dom mon dow) | `*/5 * * * *` |
| `dispatch.earliest_time` | string | Search time range start (relative or absolute) | `-5m`, `-1h`, `-24h@h` |
| `dispatch.latest_time` | string | Search time range end | `now`, `+0s` |
| `disabled` | bool | Disable the saved search | `0` (enabled), `1` (disabled) |
| `description` | string | Human-readable description | Free text |
| `schedule_window` | string | Auto-calculated window for load balancing | `auto`, `0`, `5` |
| `schedule_priority` | string | Scheduling priority | `default`, `higher`, `highest` |
| `dispatch.lookups` | bool | Enable lookups during search | `1` or `0` |
| `dispatch.max_count` | int | Max number of results | `500000` |
| `dispatch.max_time` | int | Max search runtime (seconds) | `0` (unlimited) |
| `dispatch.spawn_process` | bool | Spawn search in separate process | `1` |
| `max_concurrent` | int | Max concurrent instances | `1` |
| `realtime_schedule` | bool | Use real-time scheduling | `1` |
| `run_on_startup` | bool | Run when Splunk starts | `true` or `false` |

#### Trigger Condition Keys

| Key | Type | Description | Valid Values |
|-----|------|-------------|--------------|
| `counttype` | string | What to count for triggering. **NOT** `alert_type`. | `number of events`, `number of hosts`, `number of sources`, `custom`, `always` |
| `relation` | string | Comparison operator. **NOT** `alert_comparator`. | `greater than`, `less than`, `equal to`, `not equal to`, `drops by`, `rises by` |
| `quantity` | int | Threshold value. **NOT** `alert_threshold`. | Any integer (e.g., `0`, `5`, `100`) |
| `alert_condition` | string | Custom SPL trigger condition (used ONLY with `counttype = custom`) | `search count > 10` |
| `alert.digest_mode` | bool | `1` = trigger once per search; `0` = trigger for each result | `1` or `0` |

#### Alert Metadata Keys

| Key | Type | Description | Valid Values |
|-----|------|-------------|--------------|
| `alert.severity` | int | Severity level (1=debug, 2=info, 3=warn, 4=error, 5=critical) | `1` through `10` |
| `alert.track` | string | Track in Triggered Alerts list | `0`, `1`, `auto` |
| `alert.expires` | string | How long triggered alert entries persist | `24h`, `7d`, `30d` |
| `alert.managedBy` | string | Owner/manager identifier | Free text |

#### Throttling & Suppression Keys

| Key | Type | Description | Example Values |
|-----|------|-------------|----------------|
| `alert.suppress` | bool | Enable suppression | `0` or `1` |
| `alert.suppress.period` | string | Suppression window duration | `1h`, `30m`, `10m` |
| `alert.suppress.fields` | string | Comma-separated fields for dedup grouping | `host`, `src,dest`, `host,sourcetype` |
| `alert.suppress.group_name` | string | Named suppression group | Free text |

#### Email Action Keys

| Key | Type | Description | Example Values |
|-----|------|-------------|----------------|
| `action.email` | bool | Enable email action | `0` or `1` |
| `action.email.to` | string | Recipient email(s), comma-separated | `user@company.com` |
| `action.email.cc` | string | CC recipients | `manager@company.com` |
| `action.email.bcc` | string | BCC recipients | `archive@company.com` |
| `action.email.subject` | string | Email subject (supports tokens) | `Alert: $name$` |
| `action.email.message` | string | Email body (supports tokens) | `$name$ triggered. Results: $job.resultCount$` |
| `action.email.format` | string | Results format in email | `table`, `raw`, `csv`, `html`, `plain` |
| `action.email.inline` | bool | Include results inline | `1` or `0` |
| `action.email.sendresults` | bool | Attach results | `1` or `0` |
| `action.email.include.results_link` | bool | Include link to results | `1` or `0` |
| `action.email.include.view_link` | bool | Include link to alert | `1` or `0` |
| `action.email.include.trigger` | bool | Include trigger condition info | `1` or `0` |
| `action.email.include.trigger_time` | bool | Include trigger time | `1` or `0` |
| `action.email.priority` | int | Email priority (1=highest, 5=lowest) | `1` through `5` |
| `action.email.useNSSubject` | bool | Use namespace in subject | `1` or `0` |
| `action.email.sendpdf` | bool | Attach results as PDF | `1` or `0` |
| `action.email.sendcsv` | bool | Attach results as CSV | `1` or `0` |
| `action.email.maxresults` | int | Max results to include | `10000` |
| `action.email.maxtime` | string | Max time for email generation | `5m`, `10m` |

#### Webhook Action Keys

| Key | Type | Description | Example Values |
|-----|------|-------------|----------------|
| `action.webhook` | bool | Enable webhook action | `0` or `1` |
| `action.webhook.param.url` | string | Webhook endpoint URL | `https://hooks.example.com/alert` |

#### Log Event Action Keys

| Key | Type | Description | Example Values |
|-----|------|-------------|----------------|
| `action.logevent` | bool | Enable log event action | `0` or `1` |
| `action.logevent.param.index` | string | Target index | `alerts`, `summary` |
| `action.logevent.param.host` | string | Host value for logged event | `alerting_host` |
| `action.logevent.param.source` | string | Source value | `alert:my_alert` |
| `action.logevent.param.sourcetype` | string | Sourcetype value | `alert_log` |
| `action.logevent.param.event` | string | Event text (supports tokens) | `$name$ triggered at $trigger_time$` |

#### CSV Lookup Output Action Keys

| Key | Type | Description | Example Values |
|-----|------|-------------|----------------|
| `action.populate_lookup` | bool | Enable lookup output action | `0` or `1` |
| `action.populate_lookup.dest` | string | Target lookup name or CSV path | `my_lookup`, `etc/apps/myapp/lookups/alerts.csv` |

#### Script Action Keys (Deprecated)

| Key | Type | Description |
|-----|------|-------------|
| `action.script` | bool | Enable script action (deprecated — use custom alert actions) |
| `action.script.filename` | string | Script filename in `$SPLUNK_HOME/bin/scripts/` |

---

## BANNED Keys — NEVER Use These

These keys are common AI hallucinations that look correct but are NOT valid Splunk configuration keys. Using them causes `"Invalid key"` errors on every Splunk startup.

| BANNED Key | Error Message | Correct Key |
|------------|--------------|-------------|
| `is_scheduled` | `Invalid key in stanza: is_scheduled` | `enableSched` |
| `alert_type` | `Invalid key in stanza: alert_type` | `counttype` |
| `alert_comparator` | `Invalid key in stanza: alert_comparator` | `relation` |
| `alert_threshold` | `Invalid key in stanza: alert_threshold` | `quantity` |
| `is_visible` | `Invalid key in stanza: is_visible` | _(not applicable — use `disabled`)_ |
| `alert_condition` (as trigger type) | Misleading — exists but only for custom SPL | Use `counttype`/`relation`/`quantity` for standard triggering |

**Impact:** Across 6 affected VISTA-generated apps, these invalid keys produced **336+ "Invalid key" errors** per Splunk restart. See `/splunk-startup-error-report.md` for full audit.

---

## Multi-Line SPL in savedsearches.conf

Splunk `.conf` files terminate values at end-of-line. Multi-line SPL in a `search` key MUST end each continued line with `\` (backslash before newline).

```ini
# WRONG — lines after first are orphaned, causing "Cannot parse into key-value pair" errors
search = index=main sourcetype=foo
| stats count
| where count>0

# CORRECT — backslash continues the value
search = index=main sourcetype=foo \
| stats count \
| where count>0
```

**Validation:** After writing `savedsearches.conf`, check for orphaned pipe lines:
```bash
grep -n '^| ' default/savedsearches.conf
# Must return empty — every line starting with | must be preceded by a \-terminated line
```

---

## Cron Schedule Reference

Cron expressions consist of 5 space-separated fields:

```
┌───────────── minute (0-59)
│ ┌───────────── hour (0-23)
│ │ ┌───────────── day of month (1-31)
│ │ │ ┌───────────── month (1-12)
│ │ │ │ ┌───────────── day of week (0-6, 0=Sunday)
│ │ │ │ │
* * * * *
```

### Common Cron Patterns for Alerts

| Pattern | Cron Expression | `dispatch.earliest_time` |
|---------|----------------|--------------------------|
| Every 5 minutes | `*/5 * * * *` | `-5m` |
| Every 15 minutes | `*/15 * * * *` | `-15m` |
| Every 30 minutes | `*/30 * * * *` | `-30m` |
| Every hour | `0 * * * *` | `-1h` |
| Every 4 hours | `0 */4 * * *` | `-4h` |
| Daily at 6 AM | `0 6 * * *` | `-24h` |
| Daily at midnight | `0 0 * * *` | `-24h` |
| Weekdays at 8 AM | `0 8 * * 1-5` | `-24h` |
| First day of month at 9 AM | `0 9 1 * *` | `-1mon` |

**Best Practice:** Match the `dispatch.earliest_time` to the cron interval to avoid gaps or overlaps. If the cron runs every 5 minutes, set `dispatch.earliest_time = -5m`.

### Cron Field Formats

| Format | Meaning | Example |
|--------|---------|---------|
| `N` | Exact value | `5` = at minute 5 |
| `N,M` | Multiple values | `9,15` = at 9 AM and 3 PM |
| `I-J` | Range (inclusive) | `9-17` = 9 AM through 5 PM |
| `*` | All values | Every minute/hour/day |
| `*/N` | Every N intervals | `*/5` = every 5 minutes |
| `I-J/N` | Every N within range | `9-17/2` = every 2 hours from 9 AM to 5 PM |

---

## Trigger Conditions — How They Work

Trigger conditions act as a secondary search on the initial results. The base search runs first, then trigger conditions evaluate those results.

### Built-In Trigger Types

| `counttype` Value | What It Counts | Example Use Case |
|-------------------|----------------|------------------|
| `number of events` | Total result count | "Alert if more than 0 errors found" |
| `number of hosts` | Distinct host count | "Alert if more than 3 hosts report errors" |
| `number of sources` | Distinct source count | "Alert if errors come from multiple sources" |
| `custom` | Custom SPL condition | "Alert if any result has severity=critical" |
| `always` | Always triggers | "Always run the action (for reports)" |

### Custom Trigger Conditions

When `counttype = custom`, use `alert_condition` with an SPL expression:

```ini
[Critical Error Monitor]
search = index=_internal log_level=ERROR OR log_level=CRITICAL \
| stats count by log_level
counttype = custom
alert_condition = search log_level=CRITICAL
```

The `alert_condition` SPL runs against the results of the base `search`. If it returns any rows, the alert triggers.

### Trigger Once vs Per Result

| `alert.digest_mode` | Behavior |
|---------------------|----------|
| `1` (default) | Trigger once per search execution, regardless of result count |
| `0` | Trigger once for EACH result row that matches conditions |

---

## Alert Actions

### Email Notification

```ini
action.email = 1
action.email.to = ops@company.com
action.email.cc = manager@company.com
action.email.subject = Alert: $name$ — $job.resultCount$ results
action.email.message = The alert '$name$' triggered at $trigger_time$. \
There were $job.resultCount$ matching events.
action.email.format = table
action.email.inline = 1
action.email.include.results_link = 1
action.email.include.view_link = 1
action.email.sendresults = 1
action.email.priority = 3
```

### Available Tokens for Email

| Token | Description |
|-------|-------------|
| `$name$` | Alert/saved search name |
| `$description$` | Alert description |
| `$owner$` | Alert owner |
| `$app$` | App context |
| `$job.resultCount$` | Number of results |
| `$job.runDuration$` | Search run time |
| `$job.sid$` | Search job ID |
| `$trigger_time$` | Time the alert triggered |
| `$trigger_date$` | Date the alert triggered |
| `$results_link$` | URL to search results |
| `$view_link$` | URL to the alert |
| `$result.<fieldname>$` | Value of a specific field from the first result |

### Webhook

```ini
action.webhook = 1
action.webhook.param.url = https://hooks.slack.com/services/T00000000/B00000000/XXXX
```

### Log Event

```ini
action.logevent = 1
action.logevent.param.index = alert_events
action.logevent.param.source = alert:$name$
action.logevent.param.sourcetype = alert_log
action.logevent.param.event = Alert '$name$' triggered. Count: $job.resultCount$
```

### Output to CSV Lookup

```ini
action.populate_lookup = 1
action.populate_lookup.dest = alert_results_lookup
```

### Triggered Alerts List

```ini
alert.track = 1
alert.expires = 24h
```

---

## Throttling & Suppression

Throttling prevents alert actions from firing too frequently. The alert still triggers internally, but actions are suppressed.

```ini
alert.suppress = 1
alert.suppress.period = 1h
alert.suppress.fields = host, sourcetype
```

**How suppression works:**
1. Alert triggers and actions fire
2. During the `suppress.period`, subsequent triggers with the SAME values of `suppress.fields` are suppressed
3. Triggers with DIFFERENT field values still fire actions
4. After `suppress.period` expires, the next trigger fires actions again

### Suppression Groups

Use named suppression groups to throttle sets of similar alerts together:

```ini
alert.suppress = 1
alert.suppress.period = 30m
alert.suppress.group_name = network_alerts
```

---

## Canonical Alert Stanza Templates

### Standard Threshold Alert

```ini
[High Error Rate Detected]
description = Fires when error count exceeds threshold in the monitoring window
search = index=myindex sourcetype="my:logs" log_level=ERROR \
| stats count by host \
| where count > 10
cron_schedule = */5 * * * *
dispatch.earliest_time = -5m
dispatch.latest_time = now
enableSched = 1
disabled = 0
counttype = number of events
relation = greater than
quantity = 0
alert.severity = 4
alert.suppress = 1
alert.suppress.period = 30m
alert.suppress.fields = host
alert.track = 1
action.email = 1
action.email.to = ops@company.com
action.email.subject = [APP] High Error Rate: $name$
action.email.message = The search '$name$' found $job.resultCount$ results. $description$
```

### Absence-of-Data Alert

```ini
[No Data Received]
description = Fires when no events are received from monitored sources within the time window
search = index=myindex sourcetype="my:heartbeat" \
| stats count
cron_schedule = */15 * * * *
dispatch.earliest_time = -15m
dispatch.latest_time = now
enableSched = 1
disabled = 0
counttype = custom
alert_condition = search count < 1
alert.severity = 5
alert.suppress = 1
alert.suppress.period = 1h
alert.track = 1
action.email = 1
action.email.to = ops@company.com
action.email.subject = [CRITICAL] No data from $name$
action.email.message = No events received in the last 15 minutes. Check data sources immediately.
```

### Multi-Host Anomaly Alert

```ini
[Multi-Host Failure Detected]
description = Fires when multiple hosts simultaneously report critical errors
search = index=myindex sourcetype="my:logs" log_level=CRITICAL \
| stats dc(host) AS affected_hosts count AS total_errors \
| where affected_hosts > 3
cron_schedule = */5 * * * *
dispatch.earliest_time = -5m
dispatch.latest_time = now
enableSched = 1
disabled = 0
counttype = number of events
relation = greater than
quantity = 0
alert.severity = 5
alert.digest_mode = 1
alert.suppress = 1
alert.suppress.period = 15m
alert.track = 1
action.email = 1
action.email.to = ops@company.com, oncall@company.com
action.email.subject = [CRITICAL] Multi-host failure: $name$
action.email.message = $job.resultCount$ hosts affected. Investigate immediately.
action.email.priority = 1
```

### Webhook + Email Combined Alert

```ini
[Security Event Detected]
description = Fires on security-relevant events and notifies both Slack and email
search = index=security sourcetype="firewall:logs" action=blocked \
| stats count by src_ip, dest_ip, dest_port \
| where count > 100
cron_schedule = */5 * * * *
dispatch.earliest_time = -5m
dispatch.latest_time = now
enableSched = 1
disabled = 0
counttype = number of events
relation = greater than
quantity = 0
alert.severity = 4
alert.suppress = 1
alert.suppress.period = 1h
alert.suppress.fields = src_ip
alert.track = 1
action.email = 1
action.email.to = security@company.com
action.email.subject = [SECURITY] Blocked traffic spike: $name$
action.webhook = 1
action.webhook.param.url = https://hooks.slack.com/services/T00/B00/XXXXX
```

### OT/IoT Sensor Alert (VISTA Pattern)

```ini
[Temperature Excursion Alert]
description = Fires when temperature readings exceed defined thresholds for monitored zones
search = index=ot_sensors sourcetype="industrial:sensor" metric_name=temperature \
| lookup threshold_values_lookup sensor_type OUTPUT warn_high, alarm_high \
| eval status=case(metric_value > alarm_high, "critical", metric_value > warn_high, "warning", true(), "normal") \
| search status="critical" OR status="warning" \
| stats count by device_id, location, status, metric_value
cron_schedule = */5 * * * *
dispatch.earliest_time = -5m
dispatch.latest_time = now
enableSched = 1
disabled = 0
counttype = number of events
relation = greater than
quantity = 0
alert.severity = 4
alert.digest_mode = 1
alert.suppress = 1
alert.suppress.period = 30m
alert.suppress.fields = device_id
alert.track = 1
action.email = 1
action.email.to = ops@company.com
action.email.subject = [OT] Temperature excursion: $result.location$ - $result.device_id$
action.email.message = Device $result.device_id$ at $result.location$ reported $result.status$ temperature: $result.metric_value$
```

---

## Common Errors and Troubleshooting

### Startup Errors

| Error Message | Cause | Fix |
|--------------|-------|-----|
| `Invalid key in stanza: is_scheduled` | Using `is_scheduled` instead of `enableSched` | Replace with `enableSched = 1` |
| `Invalid key in stanza: alert_type` | Using `alert_type` instead of `counttype` | Replace with `counttype = number of events` |
| `Invalid key in stanza: alert_comparator` | Using `alert_comparator` instead of `relation` | Replace with `relation = greater than` |
| `Invalid key in stanza: alert_threshold` | Using `alert_threshold` instead of `quantity` | Replace with `quantity = 0` |
| `Cannot parse into key-value pair: \| stats count` | Multi-line SPL without `\` continuation | Add `\` to end of each continued line |
| `Duplicate stanza found` | Same `[Alert Name]` appears twice | Merge into single stanza |

### Runtime Errors

| Issue | Cause | Fix |
|-------|-------|-----|
| Alert never triggers | `enableSched = 0` or `disabled = 1` | Set `enableSched = 1` and `disabled = 0` |
| Alert triggers but no email | `action.email = 0` or missing `action.email.to` | Set `action.email = 1` and provide recipient |
| Too many alerts firing | No throttling configured | Add `alert.suppress = 1` with period and fields |
| Alert fires with 0 results | `counttype = always` or `quantity = 0` with `relation = greater than` | Set `quantity` to appropriate threshold |
| Search runs but skipped | Too many concurrent searches | Check `max_concurrent` and `dispatch.max_time` |
| Cron expression invalid | Syntax error in `cron_schedule` | Verify 5-field format: `min hour dom mon dow` |

### Validation Queries

```spl
# Check for alerts with invalid keys in internal logs
index=_internal sourcetype=splunkd "Invalid key" savedsearches.conf
| stats count by message
| sort -count

# List all scheduled alerts and their status
| rest /servicesNS/-/-/saved/searches
| search is_scheduled=1
| table title, cron_schedule, next_scheduled_time, disabled, actions

# Check for skipped scheduled searches
index=_internal sourcetype=scheduler status=skipped
| stats count by savedsearch_name, reason
| sort -count

# Check alert trigger history
index=_internal sourcetype=scheduler status=success savedsearch_name="My Alert*"
| stats count by savedsearch_name, result_count
| sort -count
```

---

## App Packaging — Alerts

### File Location

Alerts are defined in `default/savedsearches.conf` within the app:

```
<app_id>/
├── default/
│   ├── savedsearches.conf    # Alert and report definitions
│   └── alert_actions.conf    # Custom alert action definitions (if needed)
└── metadata/
    └── default.meta          # Permissions
```

### Permissions in default.meta

```ini
[savedsearches]
access = read : [ * ], write : [ admin, power ]
export = system
```

For Splunk Cloud: replace `admin` with `sc_admin`.

### AppInspect Checklist for Alerts

| Check | Requirement |
|-------|-------------|
| No invalid keys | Zero instances of `is_scheduled`, `alert_type`, `alert_comparator`, `alert_threshold` |
| No duplicate stanzas | Each `[Alert Name]` appears exactly once |
| Multi-line SPL | Every continued line ends with `\` |
| No orphaned pipe lines | No lines starting with `\|` without preceding `\` |
| All stanzas have `description` | Human-readable description for every alert |
| Cron expressions valid | 5-field format, valid ranges |
| No hardcoded credentials | No passwords, API keys, or tokens in webhook URLs |
| Email addresses appropriate | No test/personal emails in production apps |

---

## Best Practices

### Search Design for Alerts

1. **Filter early** — put `index=`, `sourcetype=`, and field filters before transforming commands
2. **Use `| stats count` for absence detection** — it always returns a row (count=0) even on empty results
3. **Avoid `| search` after `| stats`** — use `| where` instead (runs locally vs re-distributing to indexers)
4. **Keep searches simple** — complex searches increase scheduler load
5. **Use macros** for reusable filter patterns referenced by multiple alerts

### Scheduling Best Practices

1. **Match time range to schedule** — a 5-minute cron should search the last 5 minutes
2. **Stagger alert schedules** — avoid many alerts at `0 * * * *` (top of hour); spread across minutes
3. **Use `schedule_window = auto`** — lets Splunk auto-balance scheduler load
4. **Set `dispatch.max_time`** — prevent runaway searches from blocking the scheduler
5. **Use `realtime_schedule = 1`** (default) for time-sensitive alerts

### Throttling Best Practices

1. **Always configure suppression** for production alerts to prevent notification storms
2. **Use meaningful `suppress.fields`** — typically `host`, `src`, `dest`, or a combination
3. **Set `suppress.period` to at least 2x the cron interval** — prevents duplicate notifications for the same event

---

## References

- [Getting Started with Alerts — Splunk Docs](https://help.splunk.com/en/splunk-enterprise/alert-and-respond/alerting-manual/10.2/alerting-overview/getting-started-with-alerts)
- [Create Scheduled Alerts](https://help.splunk.com/en/splunk-enterprise/alert-and-respond/alerting-manual/10.2/create-alerts/create-scheduled-alerts)
- [Configure Alert Trigger Conditions](https://help.splunk.com/en/splunk-enterprise/alert-and-respond/alerting-manual/10.2/manage-alert-trigger-conditions-and-throttling/configure-alert-trigger-conditions)
- [Throttle Alerts](https://help.splunk.com/en/splunk-enterprise/alert-and-respond/alerting-manual/10.2/manage-alert-trigger-conditions-and-throttling/throttle-alerts)
- [Cron Expressions for Scheduling](https://help.splunk.com/en/splunk-enterprise/alert-and-respond/alerting-manual/10.2/create-alerts/use-cron-expressions-for-alert-scheduling)
- [Set Up Alert Actions](https://help.splunk.com/en/splunk-enterprise/alert-and-respond/alerting-manual/10.2/configure-alert-actions/set-up-alert-actions)
- [Alert Examples](https://help.splunk.com/en/splunk-enterprise/alert-and-respond/alerting-manual/10.2/alert-examples/alert-examples)
- [savedsearches.conf Reference](https://docs.splunk.com/Documentation/Splunk/latest/Admin/Savedsearchesconf)

---

## Related Skills

- **splunk-admin**: Core Splunk configuration, SPL, and knowledge objects
- **splunk-spl-commands**: SPL command reference for writing alert searches
- **splunk-cim**: CIM field mapping — alerts often query CIM-normalized data
- **splunk-lookups**: Lookup configuration — alerts may use lookup enrichment in searches
- **splunk-app-dev**: App packaging, AppInspect, and Cloud vetting
