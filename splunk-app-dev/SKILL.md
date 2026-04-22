---
name: splunk-app-dev
description: >
  Use this skill when the user wants to create, scaffold, package, or troubleshoot a Splunk app
  or add-on. Triggers include: building a Splunk app from scratch, setting up app directory
  structure, writing app.conf / commands.conf / inputs.conf / restmap.conf, creating custom search
  commands, modular inputs, alert actions, REST endpoints, packaging with SLIM or .spl,
  passing AppInspect / Splunk Cloud vetting, or any question about dev.splunk.com best practices.
---

# Splunk App Development Skill

## What this skill does

When activated, Claude acts as an expert Splunk app developer. Claude generates correct file
structures, configuration stanzas, Python code, and packaging commands based on Splunk's official
developer program standards at dev.splunk.com. Claude always produces working, AppInspect-ready
output rather than generic advice.

---

## Step 1 — Understand what the user needs before writing anything

Ask (or infer from context) these questions before generating files:

1. **App type**: Full app with UI (dashboards, nav) or headless add-on (TA / SA / DA)?
2. **Target platform**: Splunk Enterprise, Splunk Cloud, or both? (Cloud has stricter requirements)
3. **Components needed**: Which of these apply?
   - Custom search commands (Python)
   - Modular inputs (data collection)
   - Custom alert actions
   - Custom REST endpoints
   - Lookups / KV Store collections
   - Saved searches / reports / alerts
   - Dashboards (note: for dashboard creation use the splunk-dashboards skill)
4. **Splunk version target**: Minimum version affects Python SDK and conf file syntax
5. **App ID / folder name**: Must be letters, numbers, dots, underscores only — no spaces

If any of these are unclear from context, ask. Do not assume and generate the wrong structure.

---

## Step 2 — Always generate the correct directory structure first

Every Splunk app follows this layout. Generate only the directories and files the user actually needs,
but always include the three mandatory items: `default/app.conf`, `default/data/ui/nav/default.xml`,
and `metadata/default.meta`.

```
<app_id>/
├── appserver/
│   └── static/
│       ├── appIcon.png          # 36x36 (required for Splunkbase)
│       └── appIcon_2x.png      # 72x72 retina version
├── bin/                         # Python scripts: commands, inputs, alert actions, REST handlers
├── default/
│   ├── app.conf                 # REQUIRED — app identity
│   ├── commands.conf            # Custom search commands (if needed)
│   ├── alert_actions.conf       # Alert actions (if needed)
│   ├── inputs.conf              # Modular input definitions (if needed)
│   ├── props.conf               # Event processing (if needed)
│   ├── transforms.conf          # Field transforms / lookups (if needed)
│   ├── savedsearches.conf       # Saved searches / alerts (if needed)
│   ├── collections.conf         # KV Store schemas (if needed)
│   ├── restmap.conf             # REST endpoint maps (if needed)
│   ├── web.conf                 # Expose REST on port 8000 (if needed)
│   └── data/ui/
│       ├── nav/default.xml      # REQUIRED — navigation menu
│       └── views/               # Dashboard XML files
├── lib/                         # Bundled Python libraries (splunklib, etc.)
├── lookups/                     # CSV / KMZ lookup files
├── metadata/
│   └── default.meta             # REQUIRED — permissions
├── README/                      # .conf.spec and .conf.example for custom configs
└── app.manifest                 # SLIM packaging manifest
```

---

## Step 3 — Generate app.conf correctly

Always include all four stanzas. Never omit `[package]` — it causes AppInspect failures.

```ini
[install]
is_configured = 0
build = 1

[ui]
is_visible = true
label = <Human-readable App Name>

[launcher]
author = <Author or Team Name>
description = <One sentence description>
version = 1.0.0

[package]
id = <app_folder_name>
check_for_updates = true
```

**Rules:**
- `version` must be `Major.Minor.Revision` format — required for Splunkbase
- `id` must exactly match the app's folder name
- `label` must not start with "Splunk For" (AppInspect failure)
- `build` is an integer; increment it whenever static assets (CSS/JS/images) change
- For Splunk Cloud: add `[triggers]` stanzas for any custom `.conf` files

---

## Step 4 — Generate default/data/ui/nav/default.xml correctly

```xml
<nav search_view="search" color="#3C444D">
    <view name="<default_view_name>" default="true" />
    <collection label="Dashboards">
        <view source="unclassified" />
    </collection>
</nav>
```

**Rules:**
- Exactly one `default.xml` file — no other nav files are supported
- `default="true"` sets the app landing page
- `source="unclassified"` auto-populates views not explicitly listed
- `<collection>` creates dropdown menus; can be nested

---

## Step 5 — Generate metadata/default.meta correctly

```ini
[]
access = read : [ * ], write : [ admin, power ]
export = system

[views]
access = read : [ * ], write : [ admin ]
export = system

[savedsearches]
export = none
access = read : [ * ], write : [ admin, power ]
```

**Cloud-specific rule:** Replace `admin` with `sc_admin` for Splunk Cloud deployments.
**Sharing:** `export = system` shares globally; `export = none` restricts to this app only.
**Three layers:** app level `[]`, category level `[views]`, object level `[views/my_dashboard]`.

---

## Step 6 — Custom search commands (Python)

Use the **chunked protocol (V2)** always. Never use V1.

### commands.conf entry
```ini
[<commandname>]
filename = <commandname>.py
chunked = true
python.version = python3
```

### Python template — choose the right base class

| Need | Class | Method |
|------|-------|--------|
| Modify events one-by-one | `StreamingCommand` | `stream(records)` |
| Generate events from scratch | `GeneratingCommand` | `generate()` |
| Aggregate / reduce results | `ReportingCommand` | `reduce(records)` |
| Filter/reorder entire event set | `EventingCommand` | `transform(records)` |

```python
#!/usr/bin/env python
import sys, os
sys.path.insert(0, os.path.join(os.path.dirname(__file__), '..', 'lib'))

from splunklib.searchcommands import (
    dispatch, StreamingCommand, Configuration, Option, validators
)

@Configuration()
class MyCommand(StreamingCommand):
    # Define options (command arguments)
    field = Option(
        doc='Field to process',
        require=True,
        validate=validators.Fieldname()
    )

    def stream(self, records):
        for record in records:
            record['processed'] = record.get(self.field, '') + '_enriched'
            yield record

dispatch(MyCommand, sys.argv, sys.stdin, sys.stdout, __name__)
```

**Always bundle splunklib:** Copy `splunklib/` from the Python SDK into `lib/splunklib/` and
import with `sys.path.insert(0, os.path.join(os.path.dirname(__file__), '..', 'lib'))`.

**Credentials:** Never hardcode. Use `self.service.storage_passwords` to retrieve from
Splunk's encrypted credential store.

---

## Step 7 — Modular inputs (data collection)

The script implements three modes based on `sys.argv[1]`:

1. `--scheme` → print XML schema (introspection)
2. `--validate-arguments` → validate config (optional)
3. *(no arg)* → run the input, read XML config from STDIN, write events

```python
#!/usr/bin/env python
import sys, os, xml.dom.minidom

def get_scheme():
    return """<scheme>
    <title>My Input</title>
    <description>Collects data from custom source</description>
    <use_external_validation>true</use_external_validation>
    <streaming_mode>xml</streaming_mode>
    <use_single_instance>false</use_single_instance>
    <endpoint>
        <args>
            <arg name="url">
                <title>URL</title>
                <required_on_create>true</required_on_create>
            </arg>
            <arg name="interval">
                <title>Interval (seconds)</title>
                <required_on_create>false</required_on_create>
            </arg>
        </args>
    </endpoint>
</scheme>"""

def run_input(stanza_name, params, checkpoint_dir):
    # Collect data and write events
    print_xml_stream_event(stanza_name, "sourcetype", "my_data_here")

def print_xml_stream_event(stanza, sourcetype, data):
    print("<stream><event stanza=\"{}\"><sourcetype>{}</sourcetype>"
          "<data><![CDATA[{}]]></data></event></stream>".format(stanza, sourcetype, data))
    sys.stdout.flush()

if __name__ == '__main__':
    if len(sys.argv) > 1:
        if sys.argv[1] == '--scheme':
            print(get_scheme())
        elif sys.argv[1] == '--validate-arguments':
            sys.exit(0)
    else:
        # Parse XML config from STDIN and run
        pass
```

**Always use `checkpoint_dir`** to store state (last timestamp, cursor, etc.) so re-indexing
is prevented on restart. Path is passed in the STDIN config XML.

---

## Step 8 — Custom alert actions

Three required files per alert action named `<name>`:

**default/alert_actions.conf**
```ini
[<name>]
is_custom = 1
label = <Human Label>
description = <Description>
icon_path = <name>.png
payload_format = json
param.api_url =
param.auth_token =
```

**bin/<name>.py** — receives JSON payload via STDIN when called with `--execute`:
```python
import sys, json, gzip, csv

if __name__ == '__main__':
    if len(sys.argv) > 1 and sys.argv[1] == '--execute':
        payload = json.loads(sys.stdin.read())
        config = payload.get('configuration', {})
        results_file = payload.get('results_file')

        if results_file:
            with gzip.open(results_file, 'rt') as f:
                for row in csv.DictReader(f):
                    # Process each result row
                    pass
```

**default/data/ui/alerts/<name>.html** — form UI; input names follow `action.<name>.param.<key>`.

Also create `README/alert_actions.conf.spec` and `README/savedsearches.conf.spec` documenting
all custom parameters.

---

## Step 9 — Custom REST endpoints

Use `PersistentServerConnectionApplication` (not legacy `BaseRestHandler`).

**default/restmap.conf**
```ini
[script:<name>]
match                 = /<endpoint_path>
script                = <name>.py
scripttype            = persist
handler               = <name>.<ClassName>
requireAuthentication = true
output_modes          = json
passPayload           = true
capability            = admin_all_objects
```

**default/web.conf** (to expose on port 8000 Splunk Web)
```ini
[expose:<name>]
pattern = <endpoint_path>
methods = GET, POST
```

**bin/<name>.py**
```python
from splunk.persistconn.application import PersistentServerConnectionApplication
import json

class MyHandler(PersistentServerConnectionApplication):
    def __init__(self, command_line, command_arg):
        super().__init__()

    def handle(self, in_string):
        request = json.loads(in_string)
        method = request.get('method', 'GET')
        session_key = request['session']['authtoken']
        query = dict(request.get('query', []))

        # Business logic here

        return {
            'status': 200,
            'payload': json.dumps({'status': 'ok', 'data': {}}),
            'headers': {'Content-Type': 'application/json'}
        }
```

Access via: `/services/<endpoint_path>` (port 8089) or
`/splunkd/__raw/services/<endpoint_path>` (port 8000, requires CSRF token).

---

## Step 10 — AppInspect: what fails and how to prevent it

Always generate code that will pass AppInspect. Check against this list:

### Security (will FAIL)
- ❌ Hardcoded passwords, tokens, API keys anywhere in files → use `storage/passwords`
- ❌ `subprocess.Popen(..., shell=True)` → always use `shell=False`
- ❌ HTTP URLs for external calls → always use HTTPS
- ❌ File access outside the app directory boundary
- ❌ `eval()` or `exec()` on dynamic strings

### File / packaging (will FAIL)
- ❌ Shipping a `local/` directory or `metadata/local.meta`
- ❌ `.DS_Store`, `__MACOSX`, `*.pyc`, `__pycache__` in the package
- ❌ File permissions not 644 (files) / 755 (directories)
- ❌ Non-UTF-8 encoded files

### Configuration (will FAIL)
- ❌ Missing `[package] id` in app.conf
- ❌ Version not in `Major.Minor.Revision` format
- ❌ `label` starting with "Splunk For"

### Splunk Cloud-specific (will FAIL for cloud vetting)
- ❌ Using `admin` instead of `sc_admin` in metadata permissions
- ❌ `enableSched = 1` in any saved search (must be 0)
- ❌ Data model acceleration enabled
- ❌ `outputs.conf` present in the app
- ❌ Any `local/` directory content

---

## Step 11 — Packaging commands

```bash
# Install SLIM
pip install splunk-packaging-toolkit

# Generate manifest (do this once, then keep it updated)
slim generate-manifest <app_dir>/ -o <app_dir>/app.manifest

# Package
slim package <app_dir>/ -o output/

# Validate locally
slim validate output/<app_name>-*.tar.gz

# AppInspect CLI (install separately)
pip install splunk-appinspect
splunk-appinspect inspect output/<app_name>-*.tar.gz --included-tags cloud

# Run future checks (become mandatory within 3 months)
splunk-appinspect inspect output/<app_name>-*.tar.gz --included-tags future
```

A `.spl` file is just a renamed `.tar.gz`. The archive must have exactly one top-level
directory matching the app `id` in `app.conf`.

---

## Output format rules

When generating app files, Claude should:

1. **Show the full directory tree first** so the user can see what will be created
2. **Generate each file separately** with its full path as a header, e.g. `### default/app.conf`
3. **Explain any non-obvious choices** inline as comments in the file, not in a separate block
4. **Flag Cloud vs. Enterprise differences** explicitly when both platforms are targeted
5. **Always include the AppInspect checklist** at the end of any full app scaffold so the user
   knows what to verify before packaging

When generating Python code:
- Use Python 3 syntax only
- Include the `sys.path.insert` line to locate bundled `splunklib`
- Add docstrings to command classes
- Handle exceptions explicitly — don't let scripts crash silently

---

## Common patterns to recognize and handle

| User says | Claude should do |
|-----------|-----------------|
| "Build me a Splunk app for X" | Full scaffold: structure + app.conf + nav + metadata |
| "Add a custom search command" | commands.conf entry + Python class with correct base class |
| "Create a modular input" | inputs.conf + three-mode Python script + .conf.spec |
| "Build an alert action" | alert_actions.conf + Python executor + HTML form |
| "Add a REST endpoint" | restmap.conf + web.conf + PersistentServerConnectionApplication |
| "Package my app" | SLIM commands + AppInspect commands + common failure checklist |
| "Pass AppInspect / Cloud vetting" | Review against Section 10 failure list |
| "Create a TA / add-on" | Headless structure (no nav/views), focus on inputs + transforms |

---

## References
- Splunk Developer Program: https://dev.splunk.com/
- App dev docs: https://dev.splunk.com/enterprise/docs/developapps/
- AppInspect checks: https://dev.splunk.com/view/SP-CAAAE3H
- Python SDK: https://splunk-python-sdk.readthedocs.io/
