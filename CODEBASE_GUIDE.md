# CHT Core — Codebase Guide for New Contributors

Welcome! This guide is for anyone encountering the CHT Core repository for the first time. It gives you a map of the codebase, explains how the pieces fit together, and shows you where to start.

For the full contributor documentation, visit the [CHT docs site](https://docs.communityhealthtoolkit.org/community/contributing/).

---

## Table of Contents

1. [What is CHT Core?](#what-is-cht-core)
2. [High-Level Architecture](#high-level-architecture)
3. [Repository Structure](#repository-structure)
4. [Core Modules](#core-modules)
   - [webapp](#webapp)
   - [api](#api)
   - [sentinel](#sentinel)
   - [admin](#admin)
   - [shared-libs](#shared-libs)
   - [ddocs](#ddocs)
   - [config](#config)
5. [Infrastructure Components](#infrastructure-components)
6. [Key Concepts](#key-concepts)
7. [Development Environment Setup](#development-environment-setup)
8. [Building the Project](#building-the-project)
9. [Running Tests](#running-tests)
10. [Where to Go Next](#where-to-go-next)

---

## What is CHT Core?

The **Community Health Toolkit (CHT) Core Framework** is an open-source software platform that lets teams build digital health applications for community health workers. It provides:

- A **mobile-first web application** used by health workers in the field.
- A **Node.js API server** that mediates access to CouchDB.
- A **background processor (Sentinel)** that reacts to database changes and runs business-logic transitions.
- An **admin interface** for configuring the application.
- A rich set of **shared libraries** used across all modules.

The runtime database is **Apache CouchDB**, and the front-end is an **Angular** single-page application.

---

## High-Level Architecture

```
  Browser / Android app
        │
        ▼
   ┌──────────┐     ┌───────────────┐
   │  webapp  │────▶│   api (Node)  │
   └──────────┘     └──────┬────────┘
                           │  reads/writes
                           ▼
                    ┌────────────┐
                    │  CouchDB   │
                    └──────┬─────┘
                           │  _changes feed
                           ▼
                    ┌────────────┐
                    │  sentinel  │
                    └────────────┘
```

- **webapp** is the Angular SPA served by `api`.  It talks to `api` for all data.
- **api** authenticates requests, enforces authorization, applies migrations, and proxies to CouchDB.
- **sentinel** listens to the CouchDB `_changes` feed and runs _transitions_ — small configurable rules that process each document change (e.g. sending an SMS, registering a patient, scheduling follow-up tasks).
- **CouchDB** stores all application data. The `ddocs` directory contains the design documents (map/reduce views) deployed into CouchDB.

---

## Repository Structure

```
cht-core/
├── webapp/          # Angular front-end (TypeScript)
├── api/             # Node.js HTTP server
├── sentinel/        # Background task processor
├── admin/           # AngularJS admin app (configuration UI)
├── shared-libs/     # npm workspaces – libraries shared between modules
├── ddocs/           # CouchDB design documents (views, filters, etc.)
├── config/          # Reference app configurations (default, demo, covid-19)
├── tests/           # Integration and end-to-end test suites
├── scripts/         # Build scripts, CI helpers
├── nginx/           # nginx reverse-proxy config
├── haproxy/         # HAProxy config and tests
├── couchdb/         # CouchDB entrypoint and unit tests
├── release-notes/   # Per-version release notes
└── patches/         # patch-package patches applied at install time
```

---

## Core Modules

### webapp

> **Path:** `webapp/`  
> **Tech:** Angular 17, TypeScript, SCSS

The main user-facing single-page application. Key sub-directories:

| Path | Purpose |
|---|---|
| `src/ts/modules/` | Feature modules: `contacts`, `reports`, `tasks`, `messages`, `analytics` |
| `src/ts/components/` | Shared UI components (header, search bar, enketo forms, …) |
| `src/ts/services/` | Angular services for data fetching, auth, caching, etc. |
| `src/ts/effects/` | NgRx effects for side-effect management |
| `src/ts/reducers/` | NgRx reducers (application state) |
| `src/ts/selectors/` | NgRx selectors |
| `src/ts/actions/` | NgRx action creators |
| `src/ts/pipes/` | Angular pipes for formatting/translation |
| `web-components/` | Standalone `cht-form` web component for embedding Enketo forms |

**How to run the webapp dev server:**

```bash
npm run dev-api          # starts api + watches shared-libs/cht-datasource
# In another terminal, the webpack build watch is also started automatically
```

---

### api

> **Path:** `api/`  
> **Tech:** Node.js, Express

The HTTP server sits between the browser and CouchDB. It is responsible for:

- **Authentication & Authorization** – session management, role checks, data access rules.
- **Migrations** – `src/migrations/` contains scripts run once at startup to upgrade the database schema.
- **Controllers** (`src/controllers/`) – route handlers for every endpoint (forms, contacts, reports, users, export, …).
- **Services** (`src/services/`) – business logic used by controllers.
- **Static file serving** – the compiled webapp is served from `api/build/static/`.

**Environment variable required:**

```bash
export COUCH_URL='http://admin:pass@localhost:5984/medic'
node server.js
```

---

### sentinel

> **Path:** `sentinel/`  
> **Tech:** Node.js

Sentinel processes the CouchDB `_changes` feed and applies **transitions** to each document.  

Key paths:

| Path | Purpose |
|---|---|
| `src/transitions.js` | Entry point – loads and runs configured transitions |
| `src/lib/feed.js` | CouchDB changes-feed listener |
| `src/schedule/` | Cron-like scheduled tasks (e.g. message scheduling) |
| `src/lib/` | Helper utilities |

Transitions live in the shared library `shared-libs/transitions`. Each transition is a small module that decides whether it applies to a given document change and, if so, mutates the document accordingly.

```bash
export COUCH_URL='http://admin:pass@localhost:5984/medic'
node server.js
```

---

### admin

> **Path:** `admin/`  
> **Tech:** AngularJS (legacy), Bootstrap

The configuration admin UI. It is served at the `/admin/` path by the api and lets administrators manage users, app settings, forms, and translations.

---

### shared-libs

> **Path:** `shared-libs/`  
> **Tech:** Node.js (CommonJS), TypeScript (cht-datasource)

npm workspaces containing libraries shared between `api`, `sentinel`, and `webapp`. Each library is a separate package under `@medic/`.

| Library | Description |
|---|---|
| `cht-datasource` | Typed API for the CHT data model (used by webapp via the datasource service) |
| `transitions` | All Sentinel transition implementations |
| `rules-engine` | Task and target rules evaluation engine |
| `lineage` | Hydrates and minifies document lineage (parent chains) |
| `phone-number` | Phone number validation and formatting |
| `settings` | App settings loading and validation |
| `user-management` | User creation, update, and permission helpers |
| `search` | CouchDB view-based search implementation |
| `audit` | Document change tracking (audit trail) |
| `outbound` | Configurable outbound REST push for external systems |
| `logger` | Shared Winston logger |
| `infodoc` | Sidecar CouchDB record management |
| `contacts` | Contact hierarchy helpers |
| `message-utils` | SMS message utility functions |
| `translation-utils` | Nested-key translation helpers |
| `task-utils` | Task document helpers |
| `tombstone-utils` | Soft-delete tombstone management |
| `validation` | Shared validation helpers |
| `environment` | Environment variable parsing |
| `server-checks` | Startup health checks |
| `bulk-docs-utils` | CouchDB bulk_docs helpers |
| `calendar-interval` | Monthly interval calculations |
| `constants` | Shared constants |
| `view-map-utils` | Access ddoc map functions at runtime |
| `couch-request` | HTTP client wrapper for CouchDB |
| `purging-utils` | Document purge helpers |
| `registration-utils` | Patient registration helpers |
| `contact-types-utils` | Flexible contact type configuration helpers |

---

### ddocs

> **Path:** `ddocs/`

CouchDB design documents split into logical groups:

| Sub-directory | Database |
|---|---|
| `medic-db/` | Main `medic` database |
| `sentinel-db/` | `medic-sentinel` database |
| `users-db/` | `_users` database |
| `users-meta-db/` | `medic-users-meta` database |
| `logs-db/` | `medic-logs` database |

Each design document contains **map/reduce views** and **filters** used for data queries and replication.

---

### config

> **Path:** `config/`

Reference application configurations used for testing and as starting points:

| Config | Purpose |
|---|---|
| `default/` | Standard CHT reference configuration |
| `demo/` | Demo configuration for evaluations |
| `covid-19/` | COVID-19 use-case configuration |
| `standard/` | Minimal base configuration |

---

## Infrastructure Components

| Component | Path | Purpose |
|---|---|---|
| nginx | `nginx/` | TLS termination, reverse proxy to api and CouchDB |
| haproxy | `haproxy/` | Load balancing for CouchDB cluster |
| haproxy-healthcheck | `haproxy-healthcheck/` | Health-check sidecar for HAProxy |
| couchdb | `couchdb/` | CouchDB Docker entrypoint and unit tests |

---

## Key Concepts

### CouchDB and the Data Model

All application data (contacts, reports, messages, users) is stored as JSON documents in CouchDB. The database is the single source of truth and supports **offline-first** replication to PouchDB in the browser.

- [Database Schema docs](https://docs.communityhealthtoolkit.org/technical-overview/data/db-schema/)

### Contact Hierarchy

CHT organises health workers and patients into a configurable **place hierarchy** (e.g. Country → District → Health Centre → CHW Area → Patient). The `lineage` library hydrates documents with their full parent chain.

### Sentinel Transitions

A **transition** is a module in `shared-libs/transitions/src/transitions/` that:
1. Receives a CouchDB document change.
2. Decides via `filter()` whether to run.
3. Applies changes to the document via `onMatch()`.

Transitions are enabled per-deployment in `app_settings.transitions`.

### Rules Engine

The rules engine (`shared-libs/rules-engine`) evaluates **tasks** and **targets** for each contact using JavaScript rules defined in the app configuration. It runs both in the browser (webapp) and on the server (api/sentinel).

### Enketo Forms

Data collection forms are defined as **XForms** (XML) and rendered using [Enketo](https://enketo.org/). Forms are stored as CouchDB attachments and converted to HTML at runtime by `api` using `enketo-transformer`.

### NgRx State Management (webapp)

The Angular webapp uses [NgRx](https://ngrx.io/) for global state management. Data flows through: **Action → Reducer → Store → Selector → Component**, with side effects handled by **Effects**.

---

## Development Environment Setup

> The full, up-to-date guide is at: https://docs.communityhealthtoolkit.org/community/contributing/code/core/dev-environment/

Quick summary:

1. **Prerequisites:** Node.js (see `.nvmrc`), Docker, CouchDB running locally.
2. **Install dependencies:**
   ```bash
   npm ci
   ```
3. **Set environment variables:**
   ```bash
   export COUCH_URL='http://admin:pass@localhost:5984/medic'
   ```
4. **Start the dev build:**
   ```bash
   npm run build-dev          # one-time build of webapp + ddocs
   npm run dev-api            # start api with file watching
   # In another terminal:
   npm run dev-sentinel       # start sentinel with file watching
   ```

---

## Building the Project

| Command | Description |
|---|---|
| `npm run build-dev` | Full development build (webapp + ddocs + static files) |
| `npm run build-dev-watch` | Dev build with file watching |
| `npm run build` | Production CI build |
| `npm run build-ddocs` | Compile and version CouchDB design documents only |
| `npm run build-cht-form` | Build the standalone `cht-form` web component |

---

## Running Tests

### Unit Tests

```bash
npm run unit                 # all unit tests (webapp, admin, api, sentinel, shared-libs)
npm run unit-webapp          # Angular/Karma tests for webapp only
npm run unit-api             # Mocha tests for api only
npm run unit-sentinel        # Mocha tests for sentinel only
npm run unit-shared-lib      # Mocha tests for all shared-libs
npm run unit-admin           # Karma tests for admin app
```

### Integration Tests (requires local CouchDB)

```bash
npm run integration-api      # API integration tests (starts/stops CouchDB automatically)
```

### End-to-End Tests (requires full stack via Docker)

```bash
npm run wdio-local           # default WebdriverIO e2e suite
```

### Linting

```bash
npm run lint                 # ESLint + shell script checks
```

---

## Where to Go Next

| Resource | Link |
|---|---|
| Full contributor guide | https://docs.communityhealthtoolkit.org/community/contributing/ |
| Dev environment setup | https://docs.communityhealthtoolkit.org/community/contributing/code/core/dev-environment/ |
| Architecture overview | https://docs.communityhealthtoolkit.org/technical-overview/architecture/ |
| Database schema | https://docs.communityhealthtoolkit.org/technical-overview/data/db-schema/ |
| API reference | https://docs.communityhealthtoolkit.org/apps/reference/api/ |
| App settings / transitions | https://docs.communityhealthtoolkit.org/apps/reference/app-settings/transitions/ |
| Building CHT apps | https://docs.communityhealthtoolkit.org/building/ |
| Good First Issues | https://github.com/medic/cht-core/issues?q=is%3Aissue+state%3Aopen+label%3A%22Good+first+issue%22 |
| CHT Community Forum | https://forum.communityhealthtoolkit.org |

---

*This guide is a living document — if you notice anything out of date or missing, please open a pull request to improve it.*
