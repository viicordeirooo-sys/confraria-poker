# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page mobile-first web app for tracking live poker cash games ("Poker Mahau"): players, buy-ins, rebuys, rake/caixinha, cashouts, settlement (who pays whom), match history, and per-player stats. UI and all text are in Brazilian Portuguese; all money is BRL.

## Architecture

**The entire application lives in `index.html`.** There is no build step, no package manager, no bundler, no tests, no lint. Everything — React, ReactDOM, Babel Standalone, and SheetJS (XLSX) — loads from CDN `<script>` tags. JSX is transpiled in the browser at runtime via `<script type="text/babel">`.

To run/develop: open `index.html` directly in a browser (or serve the folder with any static server) and reload to see changes. There is nothing to build or install.

### State & persistence

All state lives in the single `App` component via `useState`; there is no router. Two `localStorage` keys hold everything:

- `poker-live-v2` (`KEY`) — the **single active table** (`mesa`), or absent when none.
- `poker-mahau-historico` (`HIST_KEY`) — array of finished tables, newest first.

`mesa === null` is the central switch: when null the **Dashboard** renders (new-table button + history/stats tabs); when set, the **active-table** screen renders. `persist(m)` is the one writer — it calls `setMesa` and `saveMesa` together; passing `null` clears the active table.

### Data model

```
mesa = { id, data, rake: { valor, caixinha }, jogadores: [...], cashouts: { [jogadorId]: value } }
jogador = { id, nome, buyin, buyinForma, rebuys: [{ id, valor, forma }] }
```

- **Cashouts are stored on `mesa.cashouts` keyed by player id** (not on the player object) so they survive reloads.
- **Rake has a legacy format.** New tables store `rake.valor` (absolute R$). Older records may instead have `rake.percent`. Always resolve via the existing fallback `rake.valor !== undefined ? rake.valor : Math.round(totalGeral * (rake.percent||0)/100)`. This pattern appears in both `App` (derived values) and `exportarExcel` — keep them consistent if you change it.

### Core domain logic (pure functions near the top of the file)

- `calcularAcertos(jogadores, cashouts)` — settlement algorithm. Computes each player's balance (`cashout − totalPaid`), then greedily matches largest creditors to largest debtors to minimize the number of transactions. Uses a `0.5` epsilon to ignore rounding dust.
- `exportarExcel(mesa)` — builds spreadsheet rows (per-player totals, then TOTAL / Rake / Caixinha / settlement lines) and triggers an `.xlsx` download via the global `XLSX`.
- `encerrarMesa()` (inside `App`) — snapshots the active table into history with computed `resultados`, auto-exports Excel, then clears the active table.
- `statsJogadores` (derived in `App`) — aggregates history into per-player lifetime stats. **Keyed by player name (`nome`), not id**, so renaming a player across sessions splits their stats.

## Conventions

- Constants drive the quick-pick UI: `FORMAS` (payment methods), `BUYINS`, `REBUYS`. Edit these to change the preset buttons.
- Styling is one big template-literal CSS string (`CSS`) injected via `<style>`, plus inline styles for one-offs. Dark felt-green palette; fonts `Bebas Neue` (display) and `DM Mono` (body).
- `fmtBRL`, `uid` are the shared helpers for currency formatting and id generation.
- localStorage reads/writes are wrapped in try/catch and fail silently — preserve this so a blocked/full storage never crashes the app.
