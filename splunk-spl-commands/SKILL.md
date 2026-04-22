---
name: splunk-spl-commands
description: >
  Complete SPL (Search Processing Language) command, function, and anti-pattern reference. Use this skill when:
  (1) Writing or debugging SPL queries — filtering, extraction, aggregation, transformation commands;
  (2) Using timechart, chart, or stats — understanding syntax, span, split-by, and when to use each;
  (3) Looking up statistical functions (count, dc, avg, perc95, values, list);
  (4) Using eval functions — string manipulation, math, date/time, conditional, multivalue, JSON, type checking;
  (5) Working with regular expressions in Splunk (rex, regex, named captures);
  (6) Formatting timestamps with strftime/strptime specifiers;
  (7) Combining result sets (join alternatives, append, union, appendpipe, subsearch);
  (8) Writing optimized SPL — avoiding common anti-patterns that cause errors or poor performance;
  (9) Writing dashboard SPL for Dashboard Studio or Simple XML visualizations;
  (10) Using tstats for accelerated data model queries.
---

# SPL Command & Function Reference

## Command Decision Tree

```
What do you need?
│
├── Aggregate data over time for a LINE/AREA chart?
│   └── USE: timechart span=<time> <agg_func> [by <one_field>]
│
├── Aggregate data by a NON-TIME field for a BAR/COLUMN/PIE chart?
│   └── USE: chart <agg_func> OVER <x_field> [BY <split_field>]
│       OR:  stats <agg_func> by <field1>, <field2>
│
├── Aggregate for a TABLE or SINGLE VALUE?
│   └── USE: stats <agg_func> by <fields>
│
├── Add running totals or per-event aggregates?
│   ├── Keep all rows + add aggregate column → eventstats
│   └── Running/cumulative calculation → streamstats
│
├── Fast query against an accelerated data model?
│   └── USE: tstats <agg_func> FROM datamodel=<name> WHERE ...
│
└── Generate synthetic data for testing?
    └── USE: | makeresults count=N | eval ...
```

---

## timechart — Time-Series Aggregation (Deep Dive)

### Syntax

```spl
| timechart [span=<time>] [bins=<N>] <agg-func>(<field>) [AS <alias>] [by <split-field>]
```

### What timechart Does

`timechart` creates a statistical aggregation **over the `_time` field** — the X-axis is always time. It is the ONLY command that should be used for time-series line/area charts.

### Critical Rules

| Rule | Details |
|------|---------|
| X-axis is ALWAYS `_time` | You cannot change the X-axis field. Use `chart` for non-time X-axes. |
| Only ONE `by` clause | `timechart avg(temp) by zone` is valid. `timechart avg(temp) by zone, building` is NOT. |
| `by` field becomes series | Each unique value of the `by` field creates a separate line/series in the chart. |
| `span` sets bin size | `span=5m` = 5-minute bins. If omitted, Splunk auto-calculates from the time range. |
| `span` vs `bins` | Specify one or the other. If both given, `span` wins and `bins` is ignored. |
| Default limit: top 10 | Only the top 10 `by` field values are shown. Use `limit=0` for all, or `limit=N`. |
| Cannot split on aggregated field | `timechart sum(A) by A` fails. Use `eval A1=A | timechart sum(A) by A1`. |

### span Values

| Unit | Syntax | Example |
|------|--------|---------|
| Seconds | `s`, `sec`, `secs`, `second`, `seconds` | `span=30s` |
| Minutes | `m`, `min`, `mins`, `minute`, `minutes` | `span=5m` |
| Hours | `h`, `hr`, `hrs`, `hour`, `hours` | `span=1h` |
| Days | `d`, `day`, `days` | `span=1d` |
| Weeks | `w`, `week`, `weeks` | `span=1w` |
| Months | `mon`, `month`, `months` | `span=1mon` |
| Quarters | `q`, `qtr`, `qtrs`, `quarter`, `quarters` | `span=1q` |

### Default Spans (When No span Specified)

| Time Range Picker | Auto Span |
|-------------------|-----------|
| Last 15 minutes | 10 seconds |
| Last 60 minutes | 1 minute |
| Last 4 hours | 5 minutes |
| Last 24 hours | 30 minutes |
| Last 7 days | 1 day |
| Last 30 days | 1 day |
| Previous year | 1 month |

### Correct timechart Examples

```spl
# Simple time series — event count per 5 minutes
index=myindex | timechart span=5m count

# Average temperature per 5 minutes, split by zone (one line per zone)
index=ot_sensors metric_name=temperature
| timechart span=5m avg(metric_value) by zone

# Multiple aggregations WITHOUT split-by (each becomes its own series)
index=ot_sensors device_id="motor-001"
| timechart span=5m avg(vibration_rms) AS "RMS" max(vibration_peak) AS "Peak"

# Multiple aggregations WITH split-by — ONLY ONE by clause allowed
index=ot_sensors
| timechart span=5m avg(metric_value) by metric_name

# Limit series to top 5 by total volume
index=web sourcetype=access_combined
| timechart span=1h count by uri_path limit=5

# Show ALL series (no top-N limiting)
index=ot_sensors | timechart span=5m avg(temperature) by zone limit=0

# Use where clause to filter series by spike height
index=ot_sensors | timechart span=5m max(temperature) by zone where max in top5
```

### timechart Anti-Patterns

```spl
# BAD: Two by fields — causes syntax error
| timechart span=5m avg(temp) by zone, building
# FIX: Use one by field, pre-combine the others
| eval zone_building = zone . " - " . building
| timechart span=5m avg(temp) by zone_building

# BAD: eval after timechart to add reference lines — creates flat columns, not threshold lines
| timechart span=5m avg(temp) by zone | eval Threshold=80
# FIX: Use overlay or separate data source for thresholds in Dashboard Studio
# Or use appendpipe to create a reference series:
| timechart span=5m avg(temp) AS temperature
| eval Threshold=80

# BAD: No span specified — Splunk auto-calculates, often too granular or too coarse
| timechart avg(cpu_usage) by host
# FIX: Always specify span explicitly
| timechart span=5m avg(cpu_usage) by host

# BAD: Using timechart for non-time-based aggregation
| timechart count by status
# FIX: Use stats or chart for non-time grouping
| stats count by status
| chart count by status

# BAD: Splitting by a high-cardinality field without limit
| timechart span=5m count by src_ip
# FIX: Limit to top N series
| timechart span=5m count by src_ip limit=10

# BAD: Using random() in timechart for demo data
| timechart span=5m eval(round(random() % 100, 2)) AS metric
# FIX: Use fixed values in makeresults for demo data
| makeresults count=1 | eval metric=95.3
```

---

## chart — Arbitrary X-Axis Aggregation

### Syntax

```spl
| chart <agg-func>(<field>) OVER <x-axis-field> [BY <split-field>]
| chart <agg-func>(<field>) BY <row-field> <column-field>
```

### What chart Does

`chart` creates a statistical aggregation table with an arbitrary field as the X-axis — unlike `timechart` which forces `_time`. Use `chart` for bar charts, column charts, and pie charts.

### chart vs stats

| Feature | `chart` | `stats` |
|---------|---------|---------|
| X-axis field | Specified with `OVER` or first `BY` field | No X-axis concept |
| Column-split | Optional second `BY` field pivots columns | All `by` fields are row groups |
| Output | Pivot table (rows × columns) | Flat grouped table |
| Use for | Bar/column/pie charts | Tables, single values, further processing |
| Default column limit | Top 10 | No limit |

### Correct chart Examples

```spl
# Bar chart: count by sourcetype
index=myindex | chart count OVER sourcetype

# Stacked bar: count by host, split by severity
index=myindex | chart count OVER host BY severity

# Pie chart: count by status
index=myindex | chart count BY status

# Multiple aggregations
index=myindex | chart avg(response_time) max(response_time) OVER uri_path

# Ratio via eval expression (must use AS or BY with eval)
index=myindex | chart eval(avg(size)/max(delay)) AS ratio BY host
```

---

## stats, eventstats, streamstats — Aggregation Family

### stats — Group and Aggregate

```spl
| stats <func>(<field>) [AS <alias>] [, <func>(<field>) ...] [by <field-list>]
```

Use `stats` for tables, single values, or any non-charting aggregation. Multiple `by` fields create composite group keys.

```spl
# Count events by host and sourcetype
| stats count by host, sourcetype

# Multiple aggregations in one pass
| stats count AS event_count,
        avg(response_time) AS avg_rt,
        max(response_time) AS max_rt,
        dc(user) AS unique_users
        by host

# Combine eval statements with stats
| stats count, avg(eval(if(status>=400, response_time, null()))) AS avg_error_rt by host
```

### eventstats — Aggregate Without Reducing Rows

```spl
| eventstats <func>(<field>) [AS <alias>] [by <field-list>]
```

Adds aggregate values as new fields to EVERY event without removing any rows. Essential for comparing individual events against group totals.

```spl
# Add host-level average to each event for comparison
| eventstats avg(response_time) AS host_avg by host
| eval deviation = response_time - host_avg

# Add overall count while keeping individual events
| eventstats count AS total_events
| eval pct_of_total = round(1 / total_events * 100, 2)
```

### streamstats — Running/Cumulative Calculations

```spl
| streamstats <func>(<field>) [AS <alias>] [by <field-list>] [window=<N>] [current=<bool>]
```

Calculates running statistics as each event is processed. Events MUST be sorted by `_time` first.

```spl
# Running average (moving window of 10 events)
| sort _time | streamstats window=10 avg(temperature) AS moving_avg

# Cumulative sum
| sort _time | streamstats sum(bytes) AS cumulative_bytes

# Running count per host
| sort _time | streamstats count AS events_so_far by host

# Rate of change (difference from previous)
| sort _time | streamstats current=false window=1 last(metric_value) AS prev_value
| eval change = metric_value - prev_value
```

---

## tstats — Fast Accelerated Data Model Queries

### Syntax

```spl
| tstats <func>(<datamodel_field>) FROM datamodel=<name> WHERE <constraints>
  [by <field-list>] [span=<time>]
```

### When to Use tstats

Use `tstats` instead of `stats` when querying CIM-accelerated data models. Orders of magnitude faster because it reads pre-computed summaries, not raw events.

```spl
# Count network traffic events per source
| tstats count FROM datamodel=Network_Traffic WHERE All_Traffic.action=allowed
  by All_Traffic.src
| rename All_Traffic.* AS *

# Time-series from data model (like timechart but against data model)
| tstats count FROM datamodel=Authentication WHERE nodename=Authentication
  by _time span=1h Authentication.action
| rename Authentication.* AS *

# Summaries index (non-data model but indexed fields)
| tstats count WHERE index=myindex sourcetype=mytype by host _time span=5m
```

### tstats Rules

| Rule | Details |
|------|---------|
| Requires accelerated data models OR `tsidx` | Cannot use on non-accelerated data |
| Field names are prefixed | Use `All_Traffic.src` not just `src` |
| Rename after | Use `\| rename DataModel.* AS *` for clean output |
| Supports `span=` | For time bucketing, similar to `timechart` |
| No `eval` inside tstats | Must `eval` after `tstats`, not within |

---

## Command Reference Tables

### Filtering Commands

| Command | Syntax | Description | Notes |
|---------|--------|-------------|-------|
| `search` | `search <expression>` | Filter events matching boolean expression | First command is implicit search. After a pipe, use `where` instead. |
| `where` | `where <eval-expression>` | Filter using eval expressions | Runs locally on search head — faster than `search` after transforming commands. |
| `regex` | `regex <field>=<regex>` | Filter events matching regex | Cannot add new fields — use `rex` for extraction. |
| `dedup` | `dedup [N] <field-list>` | Remove duplicate events | Keeps first N occurrences per unique combination. |
| `head` | `head <N>` | Keep first N results | Use after `sort` for top-N patterns. |
| `tail` | `tail <N>` | Keep last N results | Less common than `head`. |

### CRITICAL: `where` vs `search` After Transforming Commands

```spl
# BAD: | search after | stats re-distributes to indexers
| stats count by host | search count > 10

# GOOD: | where runs locally on search head — much faster
| stats count by host | where count > 10
```

### Data Extraction Commands

| Command | Syntax | Description |
|---------|--------|-------------|
| `rex` | `rex field=<f> "<regex>"` | Extract fields with named capture groups |
| `rex mode=sed` | `rex field=<f> mode=sed "s/old/new/g"` | In-place substitution (sed mode) |
| `spath` | `spath [input=<f>] [path=<p>] [output=<o>]` | Extract fields from JSON or XML |
| `extract` | `extract [reload=true]` | Re-run field extractions on events |
| `xmlkv` | `xmlkv` | Extract XML key-value pairs |
| `multikv` | `multikv [fields <f-list>]` | Extract from table-formatted events |

### Transformation Commands

| Command | Syntax | Description |
|---------|--------|-------------|
| `eval` | `eval <field>=<expr> [, <f2>=<expr2>]` | Calculate/create fields. Chain with commas. |
| `rename` | `rename <old> AS <new>` | Rename fields. Supports wildcards: `rename *_src AS src_*` |
| `fields` | `fields [+\|-] <field-list>` | Include (+) or exclude (-) fields. Reduces data volume early. |
| `table` | `table <field-list>` | Display only specified fields in specified order |
| `convert` | `convert <func>(<field>)` | Type conversion: `ctime`, `mktime`, `dur2sec`, `memk`, `num`, `rmcomma`, `rmunit` |
| `fillnull` | `fillnull [value=<v>] [<fields>]` | Replace null values with a default |
| `filldown` | `filldown [<fields>]` | Fill null values with previous non-null value |
| `makemv` | `makemv delim="," <field>` | Convert single-value to multivalue field |
| `mvexpand` | `mvexpand <field> [limit=<N>]` | Expand multivalue field to separate events |
| `mvcombine` | `mvcombine [delim=","] <field>` | Combine events by field into multivalue |
| `bin` / `bucket` | `bin <field> [span=<v>] [bins=<N>]` | Discretize continuous values into bins |
| `addtotals` | `addtotals [row=<bool>] [col=<bool>]` | Add row/column totals to results |
| `foreach` | `foreach <field-list> [eval <<FIELD>>_pct=...]` | Apply eval to each field matching a pattern |
| `fieldformat` | `fieldformat <field>=<format-expr>` | Format display without changing value |
| `replace` | `replace <old> WITH <new> IN <fields>` | String replacement in specified fields |

### Combining Results

| Command | Syntax | When to Use |
|---------|--------|-------------|
| `append` | `\| append [subsearch]` | Add rows from a subsearch (concatenate) |
| `appendpipe` | `\| appendpipe [run=true] [<commands>]` | Fork results, process, and append back |
| `union` | `\| union [<searches>]` | Combine multiple search results |
| `join` | `\| join [type=<t>] <field> [subsearch]` | SQL-like join — **USE SPARINGLY** (see anti-patterns) |
| `appendcols` | `\| appendcols [override=true] [subsearch]` | Add columns from a subsearch |
| `multisearch` | `\| multisearch [<searches>]` | Run multiple searches in parallel |

### Lookup Commands

| Command | Syntax | Description |
|---------|--------|-------------|
| `lookup` | `lookup <table> <field> [AS <alias>] OUTPUT <fields>` | Enrich events from a lookup table |
| `lookup` | `lookup <table> <field> [AS <alias>] OUTPUTNEW <fields>` | Like OUTPUT but won't overwrite existing fields |
| `inputlookup` | `inputlookup [append=true] <filename>` | Load lookup table as search results |
| `outputlookup` | `outputlookup [append=true] <filename>` | Write results to a lookup table |

### Reporting Commands

| Command | Syntax | Description |
|---------|--------|-------------|
| `top` | `top [limit=N] <field> [by <fields>]` | Most common values with count and percent |
| `rare` | `rare [limit=N] <field> [by <fields>]` | Least common values with count and percent |
| `contingency` | `contingency <field1> <field2>` | Cross-tabulation (contingency table) |
| `transpose` | `transpose [N] [column_name=<name>]` | Swap rows and columns |

### Data Generation Commands

| Command | Syntax | Description |
|---------|--------|-------------|
| `makeresults` | `\| makeresults [count=N] [annotate=true]` | Generate synthetic events |
| `gentimes` | `\| gentimes start=<t> end=<t> [increment=<i>]` | Generate time-range events |
| `inputcsv` | `\| inputcsv <filename>` | Read CSV file as results |

### Metadata Commands

| Command | Syntax | Description |
|---------|--------|-------------|
| `metadata` | `\| metadata type=<t> index=<i>` | Get index metadata (hosts, sources, sourcetypes) |
| `rest` | `\| rest <endpoint> [count=0]` | Query Splunk REST API endpoints |
| `dbinspect` | `\| dbinspect index=<i>` | Inspect index bucket details |

### Transaction Commands

| Command | Syntax | Description |
|---------|--------|-------------|
| `transaction` | `transaction <fields> [maxspan=<t>] [maxpause=<t>]` | Group related events into transactions |
| `concurrency` | `concurrency duration=<field>` | Calculate concurrent event count |

---

## Statistical Functions Reference

### Aggregate Functions

| Function | Description | Notes |
|----------|-------------|-------|
| `count` | Count of events | Without field: counts all events. With field: counts non-null values. |
| `count(eval(...))` | Conditional count | `count(eval(status>=400))` counts only error events |
| `dc(field)` / `distinct_count(field)` | Distinct count | Memory-expensive. Use `estdc()` for estimates on large datasets. |
| `estdc(field)` | Estimated distinct count | Much faster/lighter than `dc()` for large datasets |
| `sum(field)` | Sum of values | |
| `avg(field)` / `mean(field)` | Arithmetic mean | |
| `min(field)` | Minimum value | |
| `max(field)` | Maximum value | |
| `range(field)` | max - min | |
| `median(field)` | 50th percentile | Alias for `perc50()` |
| `mode(field)` | Most frequent value | |
| `stdev(field)` | Sample standard deviation | |
| `stdevp(field)` | Population standard deviation | |
| `var(field)` | Sample variance | |
| `varp(field)` | Population variance | |
| `sumsq(field)` | Sum of squares | |

### Percentile Functions

| Function | Description |
|----------|-------------|
| `perc<N>(field)` | Nth percentile (e.g., `perc95`, `perc99`) |
| `exactperc<N>(field)` | Exact Nth percentile (slower, more accurate) |
| `upperperc<N>(field)` | Upper Nth percentile |

### Value Collection Functions

| Function | Description | Notes |
|----------|-------------|-------|
| `values(field)` | Distinct values as multivalue | Sorted alphabetically. Memory-intensive. |
| `list(field)` | All values as multivalue (including duplicates) | Memory-intensive. Preserves order. |
| `first(field)` | First occurrence by event order | |
| `last(field)` | Last occurrence by event order | |
| `earliest(field)` | Value from earliest `_time` event | |
| `latest(field)` | Value from latest `_time` event | |
| `earliest_time(field)` | Epoch time of earliest event | |
| `latest_time(field)` | Epoch time of latest event | |

### Rate Functions (for timechart/chart)

| Function | Description |
|----------|-------------|
| `per_second(field)` | Rate per second |
| `per_minute(field)` | Rate per minute |
| `per_hour(field)` | Rate per hour |
| `per_day(field)` | Rate per day |

**Important:** Rate functions do NOT set the `span`. They convert values to a rate. You must still specify `span=` separately.

---

## Eval Functions Reference

### Conditional Functions

```spl
# if(condition, true_value, false_value)
| eval status = if(code >= 400, "error", "ok")

# case(cond1, val1, cond2, val2, ..., true(), default)
| eval severity = case(
    temp > 100, "critical",
    temp > 80, "warning",
    temp > 60, "elevated",
    true(), "normal"
  )

# coalesce(field1, field2, ...) — first non-null value
| eval user = coalesce(username, src_user, actor, "unknown")

# nullif(field, value) — returns null if field equals value
| eval clean = nullif(status, "N/A")

# validate(cond1, err1, cond2, err2, ...) — returns FIRST failing error
| eval check = validate(isnum(x), "Not a number", x > 0, "Must be positive")

# in(field, value_list) — membership test
| eval is_critical = if(status IN ("critical", "fatal", "emergency"), 1, 0)
```

### String Functions

```spl
| eval lower_name = lower(name)
| eval upper_name = upper(name)
| eval trimmed = trim(field)
| eval ltrimmed = ltrim(field, " ")
| eval rtrimmed = rtrim(field, " ")
| eval sub = substr(field, start, length)
| eval length = len(field)
| eval replaced = replace(field, "pattern", "replacement")
| eval parts = split(field, ",")
| eval joined = mvjoin(mv_field, ", ")
| eval decoded = urldecode(url_field)
| eval formatted = printf("%05d", number)
| eval concatenated = field1 . " " . field2
```

### Mathematical Functions

```spl
| eval absolute = abs(field)
| eval ceiling_val = ceiling(field)
| eval floor_val = floor(field)
| eval rounded = round(field, 2)
| eval square_root = sqrt(field)
| eval power = pow(base, exponent)
| eval logarithm = log(field, 10)
| eval natural_log = ln(field)
| eval exponential = exp(field)
| eval pi_val = pi()
| eval sign = signum(field)
| eval exact = exact(3.14 * field)
```

### Date/Time Functions

```spl
| eval current_epoch = now()
| eval time_val = time()
| eval epoch = strptime(time_string, "%Y-%m-%dT%H:%M:%S")
| eval formatted = strftime(_time, "%Y-%m-%d %H:%M:%S")
| eval relative = relative_time(now(), "-1d@d")
```

### Type Checking Functions

```spl
| eval is_null_val = isnull(field)
| eval is_notnull_val = isnotnull(field)
| eval is_numeric = isnum(field)
| eval is_integer = isint(field)
| eval is_string = isstr(field)
| eval type_name = typeof(field)
```

### Type Conversion Functions

```spl
| eval num = tonumber(string_field)
| eval num_hex = tonumber("0xFF", 16)
| eval str = tostring(num_field)
| eval duration_str = tostring(seconds, "duration")
| eval hex_str = tostring(num, "hex")
| eval commas_str = tostring(num, "commas")
```

### Multivalue Functions

```spl
| eval combined = mvappend(field1, field2)
| eval count = mvcount(mv_field)
| eval element = mvindex(mv_field, 0)
| eval last = mvindex(mv_field, -1)
| eval slice = mvindex(mv_field, 2, 5)
| eval found = mvfind(mv_field, "pattern")
| eval filtered = mvfilter(match(mv_field, "error"))
| eval sorted = mvsort(mv_field)
| eval unique = mvdedup(mv_field)
| eval sequence = mvrange(0, 10, 2)
| eval zipped = mvzip(keys, values, "=")
| eval mapped = mvmap(mv_field, upper(mv_field))
```

### JSON Functions

```spl
| eval value = json_extract(json_field, "key")
| eval nested = json_extract(json_field, "outer.inner")
| eval arr = json_array("a", "b", "c")
| eval obj = json_object("key1", val1, "key2", val2)
| eval combined = json_set(json_field, "newkey", "newvalue")
| eval valid = json_valid(field)
| eval keys = json_keys(json_field)
| eval arr_len = json_array_length(json_field, "array_key")
```

### Comparison / Boolean Functions

```spl
| eval like_match = like(field, "error%")
| eval regex_match = match(field, "^\d{3}-\d{4}$")
| eval cidr_match = cidrmatch("10.0.0.0/8", src_ip)
| eval search_match = searchmatch("error OR warning")
```

---

## Time Format Specifiers

| Specifier | Description | Example |
|-----------|-------------|---------|
| `%Y` | 4-digit year | 2026 |
| `%y` | 2-digit year | 26 |
| `%m` | Month (01-12) | 03 |
| `%b` | Abbreviated month | Mar |
| `%B` | Full month name | March |
| `%d` | Day of month (01-31) | 21 |
| `%e` | Day of month (1-31, no padding) | 21 |
| `%H` | Hour 24h (00-23) | 14 |
| `%I` | Hour 12h (01-12) | 02 |
| `%M` | Minute (00-59) | 30 |
| `%S` | Second (00-59) | 45 |
| `%3N` | Milliseconds | 123 |
| `%6N` | Microseconds | 123456 |
| `%9N` | Nanoseconds | 123456789 |
| `%p` | AM/PM | PM |
| `%z` | Timezone offset | +0000 |
| `%Z` | Timezone name | UTC |
| `%s` | Unix epoch seconds | 1710512345 |
| `%w` | Weekday (0-6, 0=Sunday) | 3 |
| `%A` | Full weekday name | Wednesday |
| `%a` | Abbreviated weekday | Wed |
| `%j` | Day of year (001-366) | 080 |
| `%+` | date(1) format | Wed Mar 21 14:30:45 UTC 2026 |

### Relative Time Modifiers

| Modifier | Description | Example |
|----------|-------------|---------|
| `s` | Seconds | `-30s` |
| `m` | Minutes | `-5m` |
| `h` | Hours | `-1h` |
| `d` | Days | `-7d` |
| `w` | Weeks | `-1w` |
| `mon` | Months | `-1mon` |
| `y` | Years | `-1y` |
| `@` | Snap to unit | `-1d@d` (snap to start of day) |
| `@w0` | Snap to Sunday | `-1w@w0` |
| `@w1` | Snap to Monday | `-1w@w1` |
| `@q` | Snap to quarter | `-1q@q` |

---

## Regular Expression Syntax

### Pattern Elements

| Pattern | Matches | Example |
|---------|---------|---------|
| `\d` | Digit [0-9] | `\d{3}` matches "123" |
| `\D` | Non-digit | |
| `\w` | Word char [a-zA-Z0-9_] | `\w+` matches "hello_123" |
| `\W` | Non-word char | |
| `\s` | Whitespace | |
| `\S` | Non-whitespace | |
| `.` | Any character (except newline) | |
| `^` | Start of string | |
| `$` | End of string | |
| `\b` | Word boundary | |

### Quantifiers

| Quantifier | Meaning | Greedy? |
|------------|---------|---------|
| `*` | 0 or more | Yes |
| `+` | 1 or more | Yes |
| `?` | 0 or 1 | Yes |
| `{n}` | Exactly n | - |
| `{n,}` | n or more | Yes |
| `{n,m}` | Between n and m | Yes |
| `*?` | 0 or more (non-greedy) | No |
| `+?` | 1 or more (non-greedy) | No |

### Groups

| Syntax | Purpose |
|--------|---------|
| `(pattern)` | Capturing group |
| `(?<name>pattern)` | Named capture group (used in SPL `rex`) |
| `(?:pattern)` | Non-capturing group |
| `(?=pattern)` | Lookahead |
| `(?!pattern)` | Negative lookahead |

### CRITICAL: PCRE2 Duplicate Named Groups

Splunk 9.x+ uses PCRE2 which does NOT allow duplicate named capture groups by default:

```ini
# BAD — causes "two named subpatterns have the same name"
| rex "src=(?<host>\S+)|source (?<host>\S+)"

# GOOD — single group with alternation inside
| rex "(?:src=|source )(?<host>\S+)"
```

### rex Examples

```spl
# Extract IP address
| rex field=_raw "(?<src_ip>\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3})"

# Extract key=value pairs
| rex field=_raw "user=(?<user>\S+)"

# Substitution mode (masking)
| rex field=message mode=sed "s/password=\S+/password=MASKED/g"

# Multiple extractions from structured log
| rex field=_raw "level=(?<log_level>\w+)\s+msg=\"(?<message>[^\"]+)\""
```

---

## SPL Anti-Patterns — Common Mistakes

### Anti-Pattern 1: `| search` After Transforming Commands

```spl
# BAD — search re-distributes to indexers, very slow
| stats count by host | search count > 10

# GOOD — where runs locally on search head
| stats count by host | where count > 10
```

### Anti-Pattern 2: Using `join` When Not Necessary

`join` silently truncates at 50,000 events with a 60-second subsearch timeout. Avoid it.

```spl
# BAD — join truncates silently
index=network | stats count BY host
| join host [search index=assets | stats latest(location) BY host]

# GOOD — OR + stats (no truncation, no timeout)
(index=network) OR (index=assets)
| stats count AS network_events, latest(location) AS location BY host

# GOOD — lookup enrichment (if assets are in a lookup)
index=network | stats count BY host
| lookup asset_lookup host OUTPUT location
```

### Anti-Pattern 3: Using `random()` in Demo/Placeholder SPL

```spl
# BAD — jittery values on every refresh, never stabilizes
| eval uptime = round(99.5 + (random() % 50) / 100, 2)

# GOOD — stable values for demo/testing
| eval uptime = 99.95
```

### Anti-Pattern 4: Separate `eval` Statements Instead of Chaining

```spl
# BAD — three separate eval passes
| eval a = lower(field1)
| eval b = upper(field2)
| eval c = a . " " . b

# GOOD — single eval with comma-separated assignments
| eval a = lower(field1), b = upper(field2), c = a . " " . b
```

### Anti-Pattern 5: Missing `coalesce()` for Lookup Fallbacks

```spl
# BAD — null value if lookup misses
| lookup asset_lookup host OUTPUT asset_name
| table host, asset_name

# GOOD — provide fallback for missing lookups
| lookup asset_lookup host OUTPUT asset_name
| eval asset_name = coalesce(asset_name, host, "unknown")
| table host, asset_name
```

### Anti-Pattern 6: Not Filtering Early

```spl
# BAD — transforms ALL events, then filters
index=* | stats count by host, sourcetype | where sourcetype="syslog"

# GOOD — filter at index time, minimize data scanned
index=myindex sourcetype=syslog | stats count by host
```

### Anti-Pattern 7: Using `timechart` When `stats` or `chart` Is Appropriate

```spl
# BAD — timechart for a pie chart (X-axis is time, not what you want)
index=myindex | timechart count by status

# GOOD for pie chart — use stats
index=myindex | stats count by status

# GOOD for bar chart — use chart
index=myindex | chart count OVER host BY status
```

### Anti-Pattern 8: Using `transaction` When `stats` Suffices

`transaction` is memory-intensive and slow. Use `stats` when you only need aggregation.

```spl
# BAD — transaction just to get session duration
| transaction session_id
| eval duration = round(duration, 2)

# GOOD — stats is orders of magnitude faster
| stats min(_time) AS start, max(_time) AS end by session_id
| eval duration = round(end - start, 2)
```

### Anti-Pattern 9: Subsearch Without `| return` or `| format`

```spl
# BAD — subsearch returns full events (limited to 10K/60s)
index=main [search index=alerts | fields src_ip]

# GOOD — use format or return for explicit value passing
index=main [search index=alerts | dedup src_ip | fields src_ip | format]

# BETTER — avoid subsearch entirely with OR
(index=main) OR (index=alerts)
| stats values(index) AS indexes by src_ip
| where mvfind(indexes, "alerts") >= 0 AND mvfind(indexes, "main") >= 0
```

### Anti-Pattern 10: Using `eval` After `timechart` for Reference Lines

```spl
# BAD — eval after timechart adds a flat column, not a threshold line
| timechart span=5m avg(temperature) by zone
| eval threshold = 80

# WHY IT'S BAD: timechart produces a multi-column table (one per zone).
# Adding eval creates a new column "threshold" that appears as a separate
# flat series. If using "by zone", the threshold column doesn't align
# with the zone-based series correctly.

# ACCEPTABLE for single-series timechart (no "by"):
| timechart span=5m avg(temperature) AS temperature
| eval threshold = 80
# This creates two columns: "temperature" and "threshold" — works as
# an overlay in a line chart.

# BEST: Use separate data sources in Dashboard Studio for reference lines
# Data Source 1: timechart query for the metric
# Data Source 2: | makeresults | eval threshold=80
```

### Anti-Pattern 11: `timechart` with Multiple `by` Fields

```spl
# BAD — syntax error: timechart only accepts ONE by field
| timechart span=5m avg(temp) by zone, building

# GOOD — concatenate fields to create a composite series label
| eval zone_building = zone . " - " . building
| timechart span=5m avg(temp) by zone_building

# ALTERNATIVE — use chart if you don't need time on X-axis
| chart avg(temp) OVER zone BY building
```

### Anti-Pattern 12: Hardcoded Time in SPL (Dashboard Queries)

```spl
# BAD — hardcoded time range ignores dashboard time picker
index=myindex earliest=-24h latest=now | timechart span=5m count

# GOOD — let the dashboard time picker control the range
index=myindex | timechart span=5m count
# (time range comes from queryParameters.earliest / queryParameters.latest)
```

---

## Dashboard SPL Patterns

### For splunk.singlevalue (KPI Panels)

```spl
# Simple count
index=myindex sourcetype="my:type" | stats count

# Latest value
index=myindex sourcetype="my:type" | stats latest(metric_value) AS value

# Percentage calculation
index=myindex sourcetype="my:type"
| stats count(eval(status="ok")) AS good, count AS total
| eval health_pct = round(good / total * 100, 1)

# Absence detection (always returns a row)
index=myindex sourcetype="my:type" | stats count
```

### For splunk.line / splunk.area (Time Series)

```spl
# Simple trend line
index=myindex | timechart span=5m count

# Multi-series by field
index=myindex | timechart span=5m avg(response_time) by host

# Multi-metric (separate lines, no split-by)
index=myindex
| timechart span=5m avg(cpu) AS "CPU %" avg(memory) AS "Memory %"

# With threshold reference line (single series only)
index=myindex | timechart span=5m avg(temperature) AS temperature
| eval warning = 80, critical = 95
```

### For splunk.column / splunk.bar (Category Charts)

```spl
# Category counts (bar chart)
index=myindex | stats count by severity | sort -count

# Stacked columns
index=myindex | chart count OVER host BY severity

# Top N categories
index=myindex | top limit=10 sourcetype
```

### For splunk.pie

```spl
# Distribution
index=myindex | stats count by status
```

### For splunk.table

```spl
# Detail table with specific columns
index=myindex | table _time, host, sourcetype, severity, message
| sort -_time

# Summary table
index=myindex | stats count, avg(response_time) AS avg_rt, max(response_time) AS max_rt by host
| sort -count
```

---

## Search Optimization Checklist

| Priority | Optimization | Why |
|----------|-------------|-----|
| 1 | Filter by `index=` and `sourcetype=` first | Eliminates buckets before scanning |
| 2 | Add field filters (`status=error`) before pipe | Reduces events sent to search head |
| 3 | Use `fields` early to drop unneeded fields | Reduces memory and network transfer |
| 4 | Use `where` not `search` after transforms | `where` runs locally |
| 5 | Use `tstats` for accelerated data models | Orders of magnitude faster |
| 6 | Chain `eval` with commas | One pass instead of multiple |
| 7 | Use `stats` instead of `transaction` | Much faster and lighter |
| 8 | Avoid `join` — use OR + stats or lookups | `join` truncates at 50K events |
| 9 | Use `estdc()` instead of `dc()` on large sets | Much less memory |
| 10 | Specify `span=` on `timechart` explicitly | Prevents auto-calc surprises |

---

## References

- [SPL Search Reference](https://help.splunk.com/en/splunk-enterprise/spl-search-reference/10.2/introduction/welcome-to-the-search-reference)
- [Command Quick Reference](https://docs.splunk.com/Documentation/Splunk/latest/SearchReference/ListOfSearchCommands)
- [timechart Command](https://docs.splunk.com/Documentation/Splunk/latest/SearchReference/Timechart)
- [chart Command](https://docs.splunk.com/Documentation/Splunk/latest/SearchReference/Chart)
- [stats Command](https://docs.splunk.com/Documentation/Splunk/latest/SearchReference/Stats)
- [Evaluation Functions](https://docs.splunk.com/Documentation/Splunk/latest/SearchReference/CommonEvalFunctions)
- [Statistical Functions](https://docs.splunk.com/Documentation/Splunk/latest/SearchReference/CommonStatsFunctions)
- [Time Format Variables](https://docs.splunk.com/Documentation/Splunk/latest/SearchReference/Commontimeformatvariables)

---

## Related Skills

- **splunk-admin**: Core Splunk configuration and data onboarding
- **splunk-alerts**: Saved searches and alert configuration in `savedsearches.conf`
- **splunk-cim**: CIM field mapping for normalized data
- **splunk-dashboards**: Dashboard Studio visualization configuration
- **splunk-lookups**: Lookup table configuration and SPL lookup commands
