# Splunk React Apps — API Reference

## @splunk/react-ui Components

### Text Input
```jsx
import Text from '@splunk/react-ui/Text';

<Text
    value={query}
    onChange={(e, { value }) => setQuery(value)}
    onKeyDown={(e) => e.key === 'Enter' && runSearch()}
    placeholder="Enter SPL query..."
    canClear
/>
```

### Button
```jsx
import Button from '@splunk/react-ui/Button';

<Button label="Run Search" appearance="primary" onClick={runSearch} />
<Button label="Cancel" appearance="secondary" onClick={cancel} />
<Button label="Delete" appearance="destructive" onClick={del} />
```

### Table
```jsx
import Table from '@splunk/react-ui/Table';

<Table stripeRows>
    <Table.Head>
        <Table.HeadCell>Name</Table.HeadCell>
        <Table.HeadCell>Value</Table.HeadCell>
    </Table.Head>
    <Table.Body>
        {rows.map((row, i) => (
            <Table.Row key={i}>
                <Table.Cell>{row.name}</Table.Cell>
                <Table.Cell>{row.value}</Table.Cell>
            </Table.Row>
        ))}
    </Table.Body>
</Table>
```

### Select / Dropdown
```jsx
import Select from '@splunk/react-ui/Select';

<Select value={selected} onChange={(e, { value }) => setSelected(value)}>
    <Select.Option label="Option A" value="a" />
    <Select.Option label="Option B" value="b" />
</Select>
```

### Multiselect
```jsx
import Multiselect from '@splunk/react-ui/Multiselect';

<Multiselect values={selectedValues} onChange={(e, { values }) => setValues(values)}>
    <Multiselect.Option label="Alpha" value="alpha" />
    <Multiselect.Option label="Beta" value="beta" />
</Multiselect>
```

### Layout
```jsx
import ColumnLayout from '@splunk/react-ui/ColumnLayout';

<ColumnLayout>
    <ColumnLayout.Row>
        <ColumnLayout.Column span={8}>
            {/* Main content */}
        </ColumnLayout.Column>
        <ColumnLayout.Column span={4}>
            {/* Sidebar */}
        </ColumnLayout.Column>
    </ColumnLayout.Row>
</ColumnLayout>
```

### ControlGroup (form labels)
```jsx
import ControlGroup from '@splunk/react-ui/ControlGroup';

<ControlGroup label="Search Query" help="Enter a valid SPL query">
    <Text value={query} onChange={handleChange} />
</ControlGroup>
```

### Tabs
```jsx
import TabBar from '@splunk/react-ui/TabBar';

<TabBar activeTabId={activeTab} onChange={(e, { selectedTabId }) => setActiveTab(selectedTabId)}>
    <TabBar.Tab label="Search" tabId="search" />
    <TabBar.Tab label="Alerts" tabId="alerts" />
</TabBar>
```

### Modal
```jsx
import Modal from '@splunk/react-ui/Modal';

<Modal open={isOpen} onRequestClose={() => setIsOpen(false)}>
    <Modal.Header title="Confirm Action" />
    <Modal.Body>Are you sure?</Modal.Body>
    <Modal.Footer>
        <Button label="Cancel" onClick={() => setIsOpen(false)} />
        <Button label="Confirm" appearance="primary" onClick={handleConfirm} />
    </Modal.Footer>
</Modal>
```

### Message / Toast
```jsx
import Message from '@splunk/react-ui/Message';

<Message type="success">Search completed successfully.</Message>
<Message type="error">Search failed: invalid query.</Message>
<Message type="warning">Results truncated to 10,000 rows.</Message>
<Message type="info">Tip: Use | head to limit results.</Message>
```

### WaitSpinner
```jsx
import WaitSpinner from '@splunk/react-ui/WaitSpinner';

{isLoading && <WaitSpinner size="medium" />}
```

### Switch / Toggle
```jsx
import Switch from '@splunk/react-ui/Switch';

<Switch
    value="timestamps"
    selected={showTimestamps}
    onClick={() => setShowTimestamps(!showTimestamps)}
    appearance="toggle"
>
    Show Timestamps
</Switch>
```

### Card
```jsx
import Card from '@splunk/react-ui/Card';
import CardLayout from '@splunk/react-ui/CardLayout';

<CardLayout cardMinWidth={300}>
    <Card>
        <Card.Header title="Seal 1" subtitle="THE LOBBY" />
        <Card.Body>Find the security override in the lobby.</Card.Body>
    </Card>
</CardLayout>
```

## @splunk/search-job Patterns

### React Hook Pattern
```jsx
import { useState, useEffect, useRef } from 'react';
import SearchJob from '@splunk/search-job';

function useSearch(spl, earliest = '0', latest = 'now') {
    const [results, setResults] = useState(null);
    const [error, setError] = useState(null);
    const [loading, setLoading] = useState(false);
    const subRef = useRef(null);

    useEffect(() => {
        if (!spl) return;

        setLoading(true);
        setError(null);

        const job = SearchJob.create({
            search: spl,
            earliest_time: earliest,
            latest_time: latest,
        });

        subRef.current = job.getResults().subscribe({
            next: (data) => {
                setResults(data);
                setLoading(false);
            },
            error: (err) => {
                setError(err);
                setLoading(false);
            },
        });

        return () => {
            if (subRef.current) subRef.current.unsubscribe();
        };
    }, [spl, earliest, latest]);

    return { results, error, loading };
}
```

### Results Data Structure
`getResults()` emits an object with:
```javascript
{
    fields: [
        { name: '_time', splitby_value: null },
        { name: 'host', splitby_value: null },
        // ...
    ],
    rows: [
        ['2024-01-01T00:00:00.000+00:00', 'server1', ...],
        ['2024-01-01T00:01:00.000+00:00', 'server2', ...],
    ],
    post_process_count: 100,
}
```

Access fields: `results.fields[i].name`
Access values: `results.rows[rowIndex][fieldIndex]`

### Converting to Objects
```javascript
function resultsToObjects(results) {
    if (!results || !results.fields || !results.rows) return [];
    const fieldNames = results.fields.map((f) => f.name);
    return results.rows.map((row) => {
        const obj = {};
        fieldNames.forEach((name, i) => { obj[name] = row[i]; });
        return obj;
    });
}
```

## @splunk/splunk-utils Detailed API

### config module
```javascript
import * as config from '@splunk/splunk-utils/config';

config.app          // Current app name (string)
config.locale       // Current locale (e.g. 'en-US')
config.CSRFToken    // CSRF token for POST/PUT/DELETE
config.username     // Current logged-in username
```

### url module
```javascript
import { createRESTURL } from '@splunk/splunk-utils/url';

// GET /servicesNS/nobody/my_app/storage/collections/data/my_kv
const url = createRESTURL('storage/collections/data/my_kv', {
    app: 'my_app',
    owner: 'nobody',
    sharing: 'app',
});

// Search jobs
const jobUrl = createRESTURL('search/jobs', { app: 'search' });
```

### fetch helpers
```javascript
import { handleResponse, handleError } from '@splunk/splunk-utils/fetch';

// handleResponse(expectedStatusCode) - returns parsed JSON on match, throws on mismatch
// handleError(userMessage) - wraps errors with user-friendly message
```

### KV Store CRUD Example
```javascript
async function getCollection(collection) {
    const url = createRESTURL(`storage/collections/data/${collection}`, {
        app: config.app, sharing: 'app',
    });
    const resp = await fetch(url, {
        credentials: 'include',
        headers: {
            'X-Splunk-Form-Key': config.CSRFToken,
            'X-Requested-With': 'XMLHttpRequest',
        },
    });
    return resp.json();
}

async function saveToCollection(collection, data) {
    const url = createRESTURL(`storage/collections/data/${collection}`, {
        app: config.app, sharing: 'app',
    });
    return fetch(url, {
        method: 'POST',
        credentials: 'include',
        headers: {
            'X-Splunk-Form-Key': config.CSRFToken,
            'X-Requested-With': 'XMLHttpRequest',
            'Content-Type': 'application/json',
        },
        body: JSON.stringify(data),
    }).then(handleResponse(201));
}
```

## Manual Setup Walkthrough

For adding React pages to an existing Splunk app without @splunk/create:

### 1. Initialize npm project in your app

```bash
cd my_splunk_app
npm init -y
npm install react react-dom styled-components @splunk/react-page @splunk/themes
npm install -D webpack webpack-cli babel-loader @babel/core @babel/preset-env @babel/preset-react
```

### 2. Create source directory

```
my_splunk_app/
├── src/
│   └── pages/
│       └── my_page/
│           └── index.jsx
├── webpack.config.js
└── package.json
```

### 3. Create entry point (`src/pages/my_page/index.jsx`)

```jsx
import React from 'react';
import layout from '@splunk/react-page';
import { SplunkThemeProvider } from '@splunk/themes';

function MyPage() {
    return <div>Hello from React inside Splunk!</div>;
}

layout(
    <SplunkThemeProvider family="enterprise" colorScheme="dark">
        <MyPage />
    </SplunkThemeProvider>,
    { pageTitle: 'My Page', hideFooter: true }
);
```

### 4. Create webpack config

```javascript
const path = require('path');

module.exports = {
    mode: 'production',
    entry: { my_page: './src/pages/my_page/index.jsx' },
    output: {
        path: path.resolve(__dirname, 'appserver/static/pages'),
        filename: '[name].js',
    },
    module: {
        rules: [{
            test: /\.jsx?$/,
            exclude: /node_modules/,
            use: {
                loader: 'babel-loader',
                options: {
                    presets: ['@babel/preset-env', '@babel/preset-react'],
                },
            },
        }],
    },
    resolve: { extensions: ['.jsx', '.js'] },
};
```

### 5. Create XML view (`default/data/ui/views/my_page.xml`)

```xml
<dashboard script="pages/my_page.js">
    <label>My Page</label>
    <row>
        <panel>
            <html>
                <div id="root"></div>
            </html>
        </panel>
    </row>
</dashboard>
```

### 6. Add to navigation (`default/data/ui/nav/default.xml`)

```xml
<nav search_view="search" color="#3C444D">
    <view name="my_page" default="true" />
</nav>
```

### 7. Build

```bash
npx webpack --mode production
```

### 8. Add build script to package.json

```json
{
    "scripts": {
        "build": "webpack --mode production",
        "dev": "webpack --mode development --watch"
    }
}
```

## Dev Tips

### Disable Splunk asset caching during development

Create `default/web.conf`:
```ini
[settings]
js_no_cache = true
cacheBytesLimit = 0
cacheEntriesLimit = 0
max_view_cache_size = 0
auto_refresh_views = 1
```

Remove this file before production deployment.

### Hot reload workaround

Splunk does not support hot module replacement. After rebuilding:
- Hard refresh: Cmd+Shift+R (Mac) / Ctrl+Shift+R (Windows)
- Or visit `https://<host>:8000/en-US/_bump`

### CSS Modules with webpack

```javascript
{
    test: /\.module\.css$/,
    use: [
        'style-loader',
        { loader: 'css-loader', options: { modules: true } },
    ],
}
```

### Peer Dependencies

When using `@splunk/react-ui` and `@splunk/visualizations`, install peer deps:
```bash
npm install react@^18 react-dom@^18 styled-components@^5
npm install @splunk/themes @splunk/visualization-context
```
