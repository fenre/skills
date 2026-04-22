---
name: splunk-react-apps
description: Build custom React pages and apps that run inside Splunk Enterprise using the Splunk UI Toolkit (SUIT). Use when creating React-based Splunk apps with @splunk/create, running SPL searches from React via @splunk/search-job, using @splunk/react-ui components, theming with @splunk/themes, building webpack bundles for Splunk, or packaging React Splunk apps as .spl files. Also use when integrating @splunk/visualizations charts in React or making REST API calls from React with @splunk/splunk-utils.
---

# Splunk React Apps (SUIT)

Build full React applications that run inside Splunk Enterprise, with complete control over the UI while leveraging Splunk's search, auth, and navigation.

## Core Packages

| Package | Purpose | Install |
|---------|---------|---------|
| `@splunk/create` | CLI scaffolder for new projects | `npx @splunk/create` |
| `@splunk/react-page` | Entry point — wraps React in Splunk layout | `npm i @splunk/react-page` |
| `@splunk/search-job` | Run SPL searches from React (RxJS) | `npm i @splunk/search-job` |
| `@splunk/react-ui` | Component library (Button, Table, Text, etc.) | `npm i @splunk/react-ui` |
| `@splunk/themes` | Theming (dark/light, enterprise/prisma) | `npm i @splunk/themes` |
| `@splunk/splunk-utils` | REST API utilities (URLs, CSRF, fetch) | `npm i @splunk/splunk-utils` |
| `@splunk/visualizations` | Charts (Line, Bar, Pie, SingleValue, etc.) | `npm i @splunk/visualizations` |
| `@splunk/webpack-configs` | Standardized webpack configs | `npm i -D @splunk/webpack-configs` |

Requirements: Node >= 22, Yarn >= 1.2

## Architecture

```
React Component → @splunk/react-page → Splunk Layout (app bar, nav, footer)
                → @splunk/search-job → Splunk REST API (search jobs)
                → @splunk/splunk-utils → Splunk REST API (KV Store, configs)
```

The React bundle is served from `appserver/static/pages/` inside a standard Splunk app. An XML view file loads the bundle.

## Quick Start: Scaffolding

### Option A: @splunk/create (monorepo)

```bash
mkdir my-project && cd my-project
npx @splunk/create
# Prompts: app name, page name, page type
yarn setup
```

Produces:
```
packages/
├── my-page/          # React page source
│   ├── src/
│   │   └── main/
│   │       └── webapp/pages/my_page/
│   │           └── index.tsx      # React entry point
│   └── package.json
└── my-splunk-app/    # Splunk app directory
    ├── default/
    │   ├── app.conf
    │   ├── data/ui/views/my_page.xml
    │   └── data/ui/nav/default.xml
    ├── appserver/static/pages/     # Built JS lands here
    └── metadata/default.meta
```

Dev commands:
```bash
yarn run start:demo     # Local React dev at http://localhost:8080
yarn link:app           # Symlink to $SPLUNK_HOME/etc/apps/
yarn start              # Watch + rebuild for Splunk
```

### Option B: Manual Setup (integrate into existing app)

When adding React pages to an existing Splunk app (like `nakatomi_heist`):

1. Create the page entry point
2. Create webpack config
3. Create the XML view file
4. Build and deploy

See [reference.md](reference.md) for the full manual setup walkthrough.

## Entry Point: @splunk/react-page

The entry point file mounts your React component into Splunk's page layout:

```jsx
import layout from '@splunk/react-page';
import { SplunkThemeProvider } from '@splunk/themes';
import MyTerminal from './MyTerminal';

layout(
    <SplunkThemeProvider family="enterprise" colorScheme="dark">
        <MyTerminal />
    </SplunkThemeProvider>,
    {
        pageTitle: 'My Terminal',
        hideFooter: true,
        layout: 'fixed',
    }
);
```

For React 18, use `import layout from '@splunk/react-page/18';`

**Options:**
- `pageTitle` — browser tab title
- `hideFooter` — hide Splunk footer (boolean)
- `layout` — `'fixed'` or default fluid

## Running SPL Searches

```jsx
import SearchJob from '@splunk/search-job';

const job = SearchJob.create({
    search: 'index=main | head 20',
    earliest_time: '-60m@m',
    latest_time: 'now',
});

// Get results (emits once when search completes)
job.getResults().subscribe({
    next: (results) => {
        // results.fields — array of field objects
        // results.rows — array of row arrays
    },
    error: (err) => console.error(err),
    complete: () => console.log('Search done'),
});

// Get progress updates (emits multiple times)
job.getProgress().subscribe((state) => {
    console.log(`${state.doneProgress * 100}% complete`);
});

// Preview results before search completes
job.getResultsPreview().subscribe((preview) => {
    // Same structure as getResults
});
```

**Context-specific search:**
```jsx
const job = SearchJob.create(
    { search: 'index=main | head 10' },
    { app: 'my_app', owner: 'admin' }
);
```

**From saved search:**
```jsx
const job = SearchJob.fromSavedSearch({
    name: 'My Saved Search',
    app: 'search',
    owner: 'admin',
});
```

Always unsubscribe when component unmounts:
```jsx
useEffect(() => {
    const sub = job.getResults().subscribe(/* ... */);
    return () => sub.unsubscribe();
}, []);
```

## REST API Calls

```jsx
import * as config from '@splunk/splunk-utils/config';
import { createRESTURL } from '@splunk/splunk-utils/url';
import { handleError, handleResponse } from '@splunk/splunk-utils/fetch';

const url = createRESTURL('storage/collections/data/my_collection', {
    app: config.app,
    sharing: 'app',
});

fetch(url, {
    method: 'GET',
    credentials: 'include',
    headers: {
        'X-Splunk-Form-Key': config.CSRFToken,
        'X-Requested-With': 'XMLHttpRequest',
        'Content-Type': 'application/json',
    },
})
    .then(handleResponse(200))
    .then((data) => { /* use data */ })
    .catch(handleError('Failed to fetch'));
```

Key exports from `@splunk/splunk-utils`:
- `config.app` — current app name
- `config.CSRFToken` — CSRF token for mutations
- `createRESTURL(path, options)` — build `/servicesNS/...` URLs
- `handleResponse(expectedStatus)` — response handler
- `handleError(message)` — error handler

## XML View File

Each React page needs an XML view file at `default/data/ui/views/<page_name>.xml`:

```xml
<dashboard script="pages/<page_name>.js">
    <label>Page Title</label>
    <row>
        <panel>
            <html>
                <div id="root"></div>
            </html>
        </panel>
    </row>
</dashboard>
```

The `script` attribute points to the webpack output in `appserver/static/`.

## Webpack Configuration

### Using @splunk/webpack-configs

```javascript
const { merge } = require('webpack-merge');
const baseConfig = require('@splunk/webpack-configs').default;
const path = require('path');

module.exports = merge(baseConfig, {
    entry: {
        my_page: path.join(__dirname, 'src/pages/my_page/index.jsx'),
    },
    output: {
        path: path.join(__dirname, '../appserver/static/pages'),
        filename: '[name].js',
    },
});
```

### Manual webpack config (no @splunk/webpack-configs)

```javascript
const path = require('path');

module.exports = {
    mode: 'production',
    entry: { my_page: './src/pages/my_page/index.jsx' },
    output: {
        path: path.resolve(__dirname, '../appserver/static/pages'),
        filename: '[name].js',
    },
    module: {
        rules: [
            {
                test: /\.[jt]sx?$/,
                exclude: /node_modules/,
                use: {
                    loader: 'babel-loader',
                    options: {
                        presets: ['@babel/preset-env', '@babel/preset-react'],
                    },
                },
            },
            { test: /\.css$/, use: ['style-loader', 'css-loader'] },
        ],
    },
    resolve: { extensions: ['.jsx', '.js', '.tsx', '.ts'] },
    externals: {
        'api/SplunkVisualizationBase': 'api/SplunkVisualizationBase',
        'api/SplunkVisualizationUtils': 'api/SplunkVisualizationUtils',
    },
};
```

## Theming

```jsx
import { SplunkThemeProvider } from '@splunk/themes';
import { variables, pick, mixins } from '@splunk/themes';
import styled from 'styled-components';

// Wrap your app
<SplunkThemeProvider family="enterprise" colorScheme="dark">
    <App />
</SplunkThemeProvider>

// Use theme variables in styled-components
const Panel = styled.div`
    ${mixins.reset()};
    background: ${variables.backgroundColorPage};
    color: ${variables.contentColorDefault};
    font-family: ${variables.fontFamilyMono};
`;
```

Families: `enterprise`, `prisma`
Color schemes: `light`, `dark`

## Using Visualizations in React

```jsx
import Line from '@splunk/visualizations/Line';
import Table from '@splunk/visualizations/Table';
import SingleValue from '@splunk/visualizations/SingleValue';

<Line
    width={800}
    height={300}
    dataSources={{ primary: myDataSource }}
/>
```

## Packaging

Build production bundle, then package as `.spl`:

```bash
# Build
cd packages/my-page && npm run build

# Package (from project root)
COPYFILE_DISABLE=1 tar czf my_app.spl \
    --exclude='node_modules' --exclude='.DS_Store' --exclude='._*' \
    my_app/
```

Or use SLIM:
```bash
pip install splunk-packaging-toolkit
slim package my_app/ -o output/
```

## App Structure (React + Splunk hybrid)

```
my_app/
├── appserver/static/
│   ├── application.css          # App-wide CSS
│   └── pages/
│       └── terminal.js          # Built React bundle
├── default/
│   ├── app.conf
│   ├── data/ui/nav/default.xml
│   └── data/ui/views/
│       ├── terminal.xml         # React page (loads terminal.js)
│       └── other_dashboard.xml  # Can still have Dashboard Studio views
├── metadata/default.meta
├── src/                         # React source (not deployed)
│   ├── pages/terminal/
│   │   └── index.jsx
│   ├── components/
│   └── styles/
├── package.json
└── webpack.config.js
```

React pages and Dashboard Studio dashboards can coexist in the same app. Navigation (`default.xml`) treats them identically.

## Additional Resources

- For detailed API patterns and component examples, see [reference.md](reference.md)
- [Splunk UI Documentation](https://splunkui.splunk.com)
- [Splunk Developer Portal](https://dev.splunk.com)
- [@splunk/create npm](https://www.npmjs.com/package/@splunk/create)
- [Examples Gallery](https://splunkui.splunk.com/Create/ExamplesGallery)
