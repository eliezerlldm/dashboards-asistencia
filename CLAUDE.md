# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repository stores **attendance dashboards** (`.html`) generated from CSV/Excel attendance files for different church zones (e.g., Zona Ecatepec, Zona Toluca). Each dashboard is a single self-contained HTML file — no build step, no dependencies installed locally.

## Generating a Dashboard

When the user uploads a CSV or Excel attendance file, follow the skill defined in `README.md` exactly:

1. **Parse** the file (CSV or `.xlsx`) using Python — deduplicate by name, keeping the earliest timestamp.
2. **Separate** Staff records (those with `'no encontrada'` in clave and `'ACREDITACIONES'` in iglesia).
3. **Compute** metrics: total, iglesia count, average entry time, 10-min blocks, 30-min slots, grade distribution.
4. **Build** a single `.html` file with hardcoded `const DATA` and `const STAFF` — no editable inputs, no localStorage.
5. **Save** to the corresponding zone folder (`zona-ecatepec/` or `zona-toluca/`) with naming pattern `dashboard_asistencia_<zona>_<mes><año>.html`.

## HTML Dashboard Structure (required sections in order)

1. Header — eyebrow (zone), title with date, pills (asistentes + iglesias), meta (inicio/fin), print button
2. Metrics cards — Total · Iglesias · Promedio entrada · Bloque pico
3. Chart: 10-min blocks — **always mixed type** (bar + line); bars colored by time zone, golden moving-average line (window=3)
4. Chart: 30-min slots — bar only
5. Chart: Grado distribution — donut (Coro/Miembro/Staff)
6. Iglesias list — clickable rows that expand to show person table (Nombre, Hora, Grado)
7. Credencial de Staff — gold section at the bottom
8. Print section — `@media print` expands all iglesia tables

## Key Rules

- **Deduplication**: always deduplicate before any processing — one name = one record (earliest).
- **Time thresholds**: retardo ≥ `inicio + 60 min`, tarde ≥ `inicio + 120 min`. Ask the user for the start time; default to 15:00 if not provided.
- **Colors**: use the CSS variables defined in `README.md` — greens (`--gd`, `--gm`, `--gl`) and gold (`--go`, `--gol`). Gradient header: `linear-gradient(90deg, #1b5e35, #2e7d52, #c9a227)`.
- **`Hrs` label**: only on metric cards, header meta, chart legends/tooltips — never in person-row Hora column.
- **Sentence case** throughout the UI (Spanish natural writing), except column labels, siglas (PDF, STAFF), and proper names.
- **CDN**: only `cdnjs.cloudflare.com` for Chart.js 4.4.1. File must work offline after first load.
- **No manipulation**: data is read-only — no `contenteditable`, no `localStorage`, no editable inputs.

## Folder Convention

```
zona-ecatepec/   → dashboards for Zona Ecatepec
zona-toluca/     → dashboards for Zona Toluca
```

File naming: `dashboard_asistencia_<zona>_<mes><año>.html`

## Delivery

After generating, mention hosting options:
- **Netlify Drop** (`netlify.com/drop`) — fastest, no account needed (expires 24 h without account)
- **GitHub Pages** — recommended when the user will publish multiple dashboards across zones
