# Architecture

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   Clinical Trial Risk Monitor                           │
│                                                                         │
│  ┌─────────────┐    ┌────────────────────────────────┐                 │
│  │  Sample Data │    │          Core Engine            │                 │
│  │  (JSON)      │───▶│  DeviationDetector             │                 │
│  │  protocol    │    │  ICH6Classifier                │                 │
│  │  patients    │    │  SiteRiskScoringEngine         │                 │
│  │  sites       │    │  CAPAReportGenerator           │                 │
│  └─────────────┘    └────────────┬───────────────────┘                 │
│                                  │                                      │
│                     ┌────────────┴──────────────┐                      │
│                     │                           │                       │
│               ┌─────▼──────┐          ┌────────▼──────┐               │
│               │ MCP Server  │          │  Next.js App  │               │
│               │  (Bob tools)│          │  (Dashboard)  │               │
│               └─────────────┘          └───────────────┘               │
└─────────────────────────────────────────────────────────────────────────┘
```

## Core Engine (`core/`)

The engine is a pure TypeScript library with no framework dependencies.

### `DeviationDetector`

Iterates over each patient's visit records and compares against the protocol specification:

1. **Visit presence check** — flags missed visits
2. **Visit window check** — flags out-of-window visits (day delta vs. scheduled day ± window)
3. **Procedure completeness** — flags any required procedure not recorded
4. **Assessment completeness** — flags any required assessment not recorded
5. **Medication checks** — banned co-medication detection + dosage range validation
6. **Eligibility checks** — inclusion/exclusion criteria verification

Each discrepancy is emitted as a `ProtocolDeviation` object, pre-classified by the ICH6Classifier.

### `ICH6Classifier`

Applies ICH E6(R2) severity rules deterministically:

| Scenario | Rule | Severity |
|---|---|---|
| Safety visit missed | Always | Major |
| Non-safety visit missed | Always | Minor |
| Visit window (safety) | Any breach | Major |
| Visit window (non-safety) | >7 days | Major |
| Visit window (non-safety) | 3–7 days | Minor |
| Visit window (non-safety) | <3 days | Administrative |
| Dosage >25% outside range | — | Major |
| Dosage 10–25% outside range | — | Minor |
| Dosage <10% outside range | — | Administrative |
| Banned co-medication | Always | Major |
| Missing procedure | Always | Minor |
| Missing assessment | Always | Administrative |

### `SiteRiskScoringEngine`

Produces a weighted composite score (0–100) per site:

```
score = 0.35 × majorDeviationRate
      + 0.20 × deviationTrend
      + 0.15 × minorDeviationRate
      + 0.15 × protocolAdherence_inverted
      + 0.10 × staffTrainingCurrency
      + 0.05 × siteExperience_inverted
```

All component scores are normalised to [0, 100] before weighting.

### `CAPAReportGenerator`

For each deviation at a site:
1. Infers root cause category (System / Process / Training / Communication) from deviation type
2. Generates corrective action templates (severity-tiered)
3. Generates preventive action templates
4. Sets ICH E6-aligned due dates (Major: 7/30 days; Minor: 14/60; Admin: 30/90)
5. Assigns responsible party and effectiveness check criteria

## MCP Server (`mcp-server/`)

Stdio transport. Registered with Bob as a local MCP server.

### Exposed Tools

| Tool | Description |
|---|---|
| `analyze_patient_deviations` | Detect + classify deviations for a patient or site |
| `score_site_risk` | Compute composite risk score for one site |
| `get_high_risk_sites` | List all sites ranked by risk (filterable by tier) |
| `generate_capa_report` | Full CAPA report for a site |
| `get_trial_summary` | High-level trial overview |

## Dashboard (`app/`)

Next.js 14 App Router. Server components load data at request time using the core engine directly (no API call overhead for SSR). Client components (charts) are separated with `"use client"`.

### REST API Endpoints

| Route | Method | Description |
|---|---|---|
| `/api/trial-summary` | GET | Full trial summary JSON |
| `/api/capa/[siteId]` | GET | CAPA report for a specific site |
