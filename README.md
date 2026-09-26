# VIGIL — AI Crime Network Analyzer

**Internal Hackathon Prototype · SIH 2026**

VIGIL is a FastAPI + spaCy + NetworkX + D3.js prototype that turns raw, multi-source
investigation records (FIR / CDR / Bank / Surveillance / Social) into an explorable
entity-relationship network with explainable-AI scoring, rule-based anomaly alerts,
a tamper-evident audit log, and demo-grade RBAC.

> ⚠️ **This is a synthetic-data, demo-grade prototype.** It is not connected to any
> real law-enforcement data source and its security controls (in-memory RBAC, demo
> Fernet key) are not production-ready. See [Important notes](#important-notes) below.

---

## Table of contents

- [Features](#features)
- [Tech stack](#tech-stack)
- [Project structure](#project-structure)
- [Setup & run](#setup--run)
- [Demo credentials](#demo-credentials)
- [App layout](#app-layout)
- [API overview](#api-overview)
- [Judge / demo walkthrough](#judge--demo-walkthrough)
- [Important notes](#important-notes)
- [Roadmap](#roadmap)

---

## Features

**Core analysis pipeline**
- spaCy NER + custom `EntityRuler` + regex extractors (phone numbers, vehicle plates)
- NetworkX weighted co-occurrence graph — degree, betweenness, community detection,
  articulation points
- Per-entity **explainable-AI** factor breakdown (centrality %, betweenness %,
  source-diversity %) — no black-box scores
- Rule-based alerts, including a **"claimed vs. actual contact mismatch"** check that
  compares an FIR's stated claim against corroborated graph edges

**Case & cross-case intelligence**
- Multi-case support (`CASE-A` / `CASE-B`) with per-case source breakdowns
- **Cross-case entity linking** — surfaces entities (phone numbers, names, vehicles…)
  shared between otherwise-unrelated cases, fully audited

**Security & governance (demo-grade)**
- RBAC: `INVESTIGATOR` / `DISTRICT_ADMIN` / `STATE_ADMIN`
- SHA-256 **hash-chained audit log** with a one-click chain-verify
- Fernet encryption/decryption demo endpoints
- Every privileged action (login, analysis run, cross-case scan, report generation,
  encryption demo) is written to the audit chain

**Interface**
- Left **navigation rail** with four views — no page reloads, no extra backend routes:
  - 🔍 **Investigation** — the original 3-column workspace (case overview, pipeline,
    D3 force-graph with type/link filters, Inspector / Key Players / Alerts /
    Timeline tabs, rule-based "Ask" box)
  - 📁 **Case Management** — all cases with record/source breakdowns, one-click case
    switching, and the cross-case scan results
  - 🛡️ **Audit & Security** — full audit log table, chain-verify, security status,
    and the Fernet encrypt/decrypt widget
  - 📄 **Reports** — generate an audited report for the active case and reopen any
    report generated earlier in the session from its in-memory snapshot
- **Light / dark theme toggle** (default: light), persisted per browser via
  `localStorage`; graph node labels automatically switch to a readable color in
  each theme
- A **signed-in user indicator** at the bottom of the nav rail (avatar + name/role)
  with **Switch user** and **Log out** actions
- Anonymous, read-only demo access is allowed by default for convenience — sign in
  as `district_admin` or higher to unlock the Audit & Security view

## Tech stack

| Layer      | Technology                                             |
|------------|---------------------------------------------------------|
| Backend    | FastAPI, Pydantic                                        |
| NLP        | spaCy (`en_core_web_sm`) + `EntityRuler` + regex          |
| Graph      | NetworkX                                                  |
| Security   | In-memory demo RBAC, SHA-256 hash chain, Fernet (`cryptography`) |
| Frontend   | Vanilla JS + D3.js (no framework, no build step)           |

## Project structure

```
.
├── main.py            # FastAPI app: NLP, graph analytics, RBAC, audit chain, all API routes
├── index.html          # Single-file SPA frontend (nav rail + 4 views, D3 graph, theming)
├── requirements.txt    # Python dependencies
└── README.md
```

## Setup & run

```bash
python -m venv venv

# Windows
venv\Scripts\activate
# macOS / Linux
source venv/bin/activate

pip install -r requirements.txt
python -m spacy download en_core_web_sm

uvicorn main:app --reload
```

Open **http://127.0.0.1:8000**

## Demo credentials

All demo accounts use the password `vigil123`.

| Username         | Role            |
|------------------|-----------------|
| `investigator`   | `INVESTIGATOR`  |
| `district_admin` | `DISTRICT_ADMIN`|
| `state_admin`    | `STATE_ADMIN`   |

The dashboard also allows **anonymous read access** to the synthetic demo case for
quick evaluation — sign in only when you want to exercise RBAC-gated features
(Audit & Security view, encryption demo) or generate an attributed audit trail.

## App layout

```
┌─────────────────────────────── VIGIL header (stats · theme toggle) ───────────────────────────────┐
│ 🔍 │                                                                                                 │
│ 📁 │                              active view content                                                │
│ 🛡️ │                     (Investigation / Case Mgmt / Audit / Reports)                                │
│ 📄 │                                                                                                 │
│ 👤 │  ← signed-in user avatar (click → Switch user / Log out)                                        │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘
```

Switching a rail item swaps the visible view client-side — no reload, and data is
only re-fetched the first time a view is opened (or when it needs to reflect a
state change, e.g. after switching cases or logging in/out).

## API overview

| Method | Route                          | Auth               | Purpose                                   |
|--------|---------------------------------|---------------------|--------------------------------------------|
| POST   | `/api/login`                    | —                    | Issue a demo bearer token                  |
| GET    | `/api/me`                       | any                  | Current user info                          |
| GET    | `/api/cases`                    | any                  | List available cases                       |
| GET    | `/api/case-summary`             | any                  | Case title, objective, source breakdown    |
| GET    | `/api/sample-data`              | any                  | Records for a case                         |
| POST   | `/api/extract`                  | any                  | Entity extraction                          |
| POST   | `/api/analyze`                  | any                  | Full graph analysis                        |
| POST   | `/api/cross-case-links`         | any                  | Entities shared across cases               |
| POST   | `/api/report`                   | any                  | Log an audited `REPORT_GENERATED` event    |
| GET    | `/api/audit`                    | `DISTRICT_ADMIN`+    | Audit log entries                          |
| GET    | `/api/audit/verify`             | `DISTRICT_ADMIN`+    | Verify the hash chain                      |
| POST   | `/api/security/encrypt-demo`    | `DISTRICT_ADMIN`+    | Fernet-encrypt a demo value                |
| POST   | `/api/security/decrypt-demo`    | `DISTRICT_ADMIN`+    | Fernet-decrypt a demo value                |
| GET    | `/api/security/status`          | any                  | Security feature status                    |

## Judge / demo walkthrough

1. Open the app (**Investigation** view loads by default) and click **▶ RUN FULL
   INVESTIGATION**.
2. Review entity extraction counts, then the generated network graph.
3. Click the highest-influence node — check source coverage, linked evidence, and
   the **"Why this score?"** explainability bars in the Inspector tab.
4. Open the **Alerts** tab — point out the "claimed vs. actual contact mismatch"
   alert alongside the classic uncorroborated-accusation one.
5. Switch to **📁 Case Management**, switch the active case to `CASE-B`, run the
   investigation again, then run the **cross-case scan** to show the entity shared
   with `CASE-A`.
6. Open **📄 Reports** and generate a report — a printable investigation summary
   opens in a new tab; reopen it later from the session history.
7. Sign in as `district_admin` (avatar at the bottom of the nav rail → **Switch
   user**) and open **🛡️ Audit & Security** — show the hash-chain verification and
   the full audit table, including the cross-case scan and report events just
   generated.
8. Toggle the header's light/dark theme button to show the UI adapts, including
   graph label contrast.
9. Mention the roadmap: Neo4j, multilingual NER, trained anomaly detection.

## Important notes

This remains a **synthetic-data prototype**. Do not connect it to real
law-enforcement data without proper identity/key management, encrypted storage,
TLS, durable audit persistence, data governance, testing, and authorization
controls.

## Roadmap

- Neo4j-backed graph storage
- Multilingual NER
- Trained (non-rule-based) anomaly detection model
