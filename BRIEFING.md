# Bozas Quick Entry — Master Briefing

> **Read this file first** before starting any work on the project.

## What you are building

A **voice-driven quick entry tool for Pylon ERP** that allows the Bozas team (3 people) to log work performed at client sites using natural Greek voice commands from their mobile phones. The goal is to ensure no billable work is forgotten before invoicing.

**Example use case:** A technician finishes a job at a client site, opens the app on their phone, taps the microphone button, says *"Βάλε στον Καραμήτρο 3 ώρες αλλαγή φόρμας και εγκατάσταση POS"*, confirms the parsed details, and the entry is created in Pylon as a **customer order** (παραγγελία πώλησης) for later invoicing.

This is **not** a generic SaaS — it's a focused internal tool first, designed with config-driven architecture so it can be productized later for other Pylon partners.

## Owner context

- **Owner:** Stavros Bozas, certified Epsilon Net partner (Pylon)
- **Company:** Μπόζας (Bozas), Volos, Greece
- **Team size:** 3 users initially
- **Owner's role:** Technical systems designer and AI orchestrator — uses Claude Code for implementation, claude.ai for architecture
- **Language:** All UI text in Greek; communication with the owner in Greek; code/comments in English

## Current Status

Last updated 2026-05-18.

### What's been built

- **Foundation** (`ad57f03`) — Next.js 16 App Router scaffold (TS, Tailwind v4, ESLint v9), folder layout per this brief, config loader (`src/lib/config/index.ts`) with fail-fast validation, `config/config.example.json` with placeholders, `.gitignore` extensions for `config/config.json` and `data/`.

- **Pylon API client** (`src/lib/pylon/*`) — production-ready over native fetch. Built up across `90ee776`, `546a0f7`, `f4a60df`. Modules:
  - `client.ts` — `login()`, `getData(entityCode, options?)`, `postData(entityCode, data)`. 30 s `AbortController` timeout. In-process cookie singleton. Outer envelope validation (`Status`, `Error`, `ErrorCode`) before unwrapping the double-encoded `Result`. Inner defense-in-depth check. Cookie refreshed from every successful response. One-shot re-login retry on session-shaped errors.
  - `types.ts` — typed request/response shapes; `PylonGetDataResponse<TTables = Record<string, PylonRow[]>>` matches the `Data` named-tables envelope; `PylonLoginResponse` carries the observed inner fields (`cookie`, `username`, `administrator`, `guest`, `created`, `expired`).
  - `errors.ts` — `PylonError` class with `kind: 'network' | 'http' | 'pylon'`, `status`, `messages`, `body`, `cause`.
  - `rows.ts` — `getRows<T>(response, tableName)` helper that resolves `response.Data[tableName] ?? []` (handles `BuildResultSet` omitting empty tables).

- **Diagnostics & probes** (`scripts/`) — `npm run test-config` (full elogin + getdata sanity), `npm run probe-getdata` (raw `Items` payload dump), `npm run probe-customer-search` (CustomerSearch end-to-end). All run via `tsx`.

- **Customer search API** (`src/app/api/customers/search/route.ts`) — server-side POST endpoint, validates `{ search }`, runs progressive shortening (see below), returns `{ customers, searchedTerm }`. Bad body → 400, `PylonError` → 502, unexpected → 500. Pylon credentials never reach the browser.

- **UI** (`src/components/*`, `src/app/page.tsx`) — mobile-first Greek interface:
  - `QuickEntry.tsx` (client) — parent owner of `transcript` and `selectedCustomer` state. Καθαρισμός resets both.
  - `VoiceCapture.tsx` (client, controlled) — `value` / `onChange` / `onClear` props. Web Speech API at `el-GR`, `continuous=false`, `interimResults=true`. Internal `valueRef` / `onChangeRef` so the once-installed recognition handlers always see current props without re-installing per render. Animated pulsing mic button while listening; `aria-live` status; full keyboard focus rings; Greek error mapping for every SpeechRecognition error code (`not-allowed`, `no-speech`, `audio-capture`, `network`, etc.).
  - `CustomerSearchPanel.tsx` (client) — explicit "Αναζήτηση" button (no auto-search — UX choice documented in the file). Stale-vs-current visibility derived from `state.requestedTerm === current trimmed term` at render time, so no prop-syncing `useEffect` (sidesteps Next.js 16's `set-state-in-effect` lint rule cleanly). In-flight request id guards against out-of-order responses. Caption above the result list: `Βρέθηκαν N πελάτες για «term»` with singular/plural verb + noun agreement. Selected-customer card with "Αλλαγή".
  - `src/types/speech.d.ts` — ambient declarations for `SpeechRecognition` (not yet in standard DOM types) and `Window` augmentation for the unprefixed + `webkit` constructors.
  - `src/types/customer.ts` — `CustomerSearchRow` extending `PylonRow`.

- **Cloudflare Tunnel dev compat** (`8ff8931`) — `next.config.ts` ships `allowedDevOrigins: ["*.trycloudflare.com", "*.cloudflare.com"]` (top-level in v16, not under `experimental`) so HMR + dev-only endpoints accept requests when the owner tests the UI from a phone over Cloudflare Tunnel.

### What works end-to-end (verified locally and against real Pylon)

The owner can:
1. Open the page from a mobile browser via Cloudflare Tunnel.
2. Tap the mic and speak Greek.
3. See the transcript live in an editable textarea (with manual edit fallback).
4. Tap "Αναζήτηση".
5. The whole transcript is sent to `/api/customers/search`, the server splits it into words, sorts longest-first, and progressively shortens each word until Pylon's CustomerSearch returns a hit.
6. Matching customers render as a tappable list with a "Βρέθηκαν N πελάτες για «<matched substring>»" caption.
7. Tap a customer → selected-customer card with "Αλλαγή" to reset.

### What's not built yet

See "Pending tasks" at the bottom for the full list. Headline missing pieces: NLP parser, SalesOrders write path, audit DB, Cloudflare Access integration, Docker packaging, CLI setup wizard.

## Architecture (decided)

```
Mobile browsers (3 users)
        ↓ HTTPS via Cloudflare Tunnel
        ↓ (auth via Cloudflare Access)
Web App (Next.js + TypeScript)
        ↓ HTTP REST (JSON)
Pylon Connectivity API (http://pylon-server:7024/exesjson/)
        ↓
SQL Server (Pylon DB)
```

**Key principle:** the web app never touches the Pylon database directly. All interaction is via the official Connectivity API.

## Tech stack (decided)

| Layer | Choice | Status |
|-------|--------|--------|
| Framework | Next.js 16 with App Router | ✅ in (16.2.6) |
| Language | TypeScript | ✅ |
| UI | Tailwind CSS v4 (`@tailwindcss/postcss`) | ✅ |
| Local DB | SQLite (better-sqlite3) for audit log | ⏳ pending |
| HTTP | Native fetch | ✅ |
| Voice | Web Speech API (browser-native, free, Greek support) | ✅ |
| NLP | Strategy pattern: RegexParser first, LLMParser pluggable later | ⏳ pending |
| Container | Docker (multi-arch if feasible) | ⏳ pending |
| Remote access | Cloudflare Tunnel | dev: ad-hoc `*.trycloudflare.com`; prod: pending |
| Authentication | Cloudflare Access (no custom login) — reads `Cf-Access-Authenticated-User-Email` header | ⏳ pending |
| Script runner (CLI tools) | `tsx` | ✅ |

## Pylon Integration

### Connectivity API basics

- Base URL: `http://<pylon-server>:7024/exesjson/`
- Endpoints: `elogin`, `getdata`, `postdata` (all POST, JSON in + JSON out)
- Auth: session cookie returned by `elogin`, used in subsequent calls
- Cookie lifetime: ~30 minutes (observed from `expired - created` in the login payload)

### Wire-level details (observed)

Every response is wrapped in a uniform outer envelope:

```json
{
  "Status": "OK" | "ERROR",
  "Result": "<JSON-encoded string>" | null,
  "Error": null | "<server message>",
  "ErrorCode": "" | "<code>",
  "Benchmark": <number>
}
```

- **`Status` is authoritative.** On `"ERROR"`, `Result` is null and `Error` carries the message (often Greek). The client validates the outer envelope *before* unwrapping `Result` — otherwise server errors slip through silently and the call returns `{}`. Inner-payload checks remain as defense-in-depth.
- **`Result` is double-encoded** — a JSON-string containing the real payload. Client unwraps transparently via `unwrapResult()`.

**Inner payload — `elogin`:**
```json
{ "cookie": "...", "username": "admin", "administrator": 1, "guest": 0,
  "created": "2026-05-15T04:42:45.125", "expired": "2026-05-15T05:12:45.125" }
```

**Inner payload — `getdata`:**
```json
{ "Cookie": "<refreshed cookie>", "Data": { "<TableName>": [ row1, row2, ... ] }, "Messages": [] }
```

- `Cookie` is **refreshed on every successful response** (login, getdata, postdata). The client captures it via `captureCookieFromResponse()` so the cached value tracks the server's rolling cookie.
- `Data` is keyed by the SQL alias each named query was given (e.g. `Data.Items`, `Data.Customers`). Pylon's `sc$APIContext.BuildResultSet` omits empty tables, so a missing key means "zero rows" and is normal — `getRows(resp, "Items")` returns `[]`.
- Row column names are **UPPERCASE**, matching the SQL aliases (e.g. `ITEMID`, `CSTMNAME`).
- `Messages` is `[]` on success.

**Inner payload — `postdata`:** not yet observed in this codebase. Typed defensively as the same `{ Cookie, Data, Messages }` envelope with a code-level note flagging the shape as unverified — will be locked down when SalesOrders is exercised.

### Per-script package-size caps

Different scripts enforce different `packagesize` limits server-side. The client defaults to `packagesize: 2000`, which works for `Items` but is rejected by tighter scripts like `CustomerSearch` (caps around 50). Always pass an explicit `packageSize` for scripts you're not sure about; the API surfaces the rejection cleanly via the outer envelope.

### Cookie expiry detection

The exact Pylon signal for an expired cookie is still undocumented (see questions.md). The client uses a heuristic: re-login + retry once when the response is HTTP 401/403 or the error message / body / `Messages` contains (case-insensitive) `cookie`, `session`, `expired`, `not logged`, `unauthor`, or `login` + `again`. Tighten when we see a real expiry in the wild.

### Credentials

These come from the Pylon API Template. Current dev template:

- **API Code:** UOCUQZNWVDBG7OA
- **Username / Password:** provided via `config.json`
- **Application name:** `Hercules.MyPylonCommercial` (confirm with owner)
- **Database alias:** provided via `config.json`

### Pylon API Scripts we use

**1. CustomerSearch — IMPLEMENTED**

Get-type script in the API template that:
- Accepts `@search` parameter (raw text, Greek or Latin).
- Pylon transliterates via `sc$String.SoundexConvertSqlLike(searchText)`.
- Joins `HECUSTOMERS ⋈ HETRADERS ⋈ HECONTACTS`, filters by `HECOMPID = @$System$CurCompany` and `HEACTIVE = 1`, runs `LIKE '%<soundex>%'` against `HENAMESOUNDEX`.
- Backed by a named query `CustomerSearchQry` in the API template (separate Pylon object from the script — must exist or the script throws at line 20).
- Returns rows in `Data.Customers` with columns `CSTMID, CSTMCODE, CSTMNAME, CSTMTIN, TRDRID`.

**Client-side strategy: progressive shortening.** ASR (especially Greek) frequently mis-hears word endings (inflections). Because `CustomerSearch` is a soundex substring `LIKE`, a prefix of the root word is sufficient. The API route `/api/customers/search`:

1. Normalises whitespace, splits the input on space.
2. Sorts the words **longest-first** (longer = more specific = fewer false hits).
3. For each word, walks prefixes from full length down to **4 chars** (`MIN_TERM_LENGTH`), calling Pylon for each.
4. Returns the first non-empty result with `{ customers, searchedTerm }` where `searchedTerm` is the substring that actually matched.
5. Returns `{ customers: [], searchedTerm: "" }` if nothing matched anywhere.

The 4-char floor avoids returning half the customer base on 1-2 char queries. A `PylonError` mid-loop aborts the whole request and propagates as 502 — no silent error swallowing inside the loop.

**Known pathological case:** garbage Latin input (e.g. `"xxxxxxxxxxxxx"`) returns the full 50-row package. The soundex of unpronounceable Latin likely degenerates to an empty/trivial token, so `LIKE '%X%'` matches every row. Real ASR output never triggers this (always emits Greek words or sensible Latin), so it's not blocking — but the future NLP layer should strip out content-free tokens (numbers, time words like `ωρες`) before they reach the search.

**2. SalesOrders — PENDING**

The existing `SalesOrders` script in template UOCUQZNWVDBG7OA:
- Accepts payload with `CstmID, SeriesCode, CsbrName, PmmtCode, Lines[]`
- Creates a Sales Order entry (`CmdtType = 2`, `SourceType = 9`)
- Returns `{ EntityId, CustomerId, Messages }`

**Modifications needed (still to do on the Pylon side):**
- Read `CenlComments` per line → `line.SetValue("Comment", ...)`
- Read `CenlItemDescription` per line → for display override on printouts
- Change behaviour: do NOT create a new customer if not found; return error instead

### Database structure (DO NOT GUESS column names — ask owner)

Known so far:
- `heCustomers`: `heID, heCode, heName, heNameSoundex, heTrdrID, heCompID, heActive`
- `heTraders`: `heID, heCntcID`
- `heContacts`: `heID, heTIN`
- `heTraderBranches`: `heID, heTrdrID, heName` (most have one row for "Έδρα", some have more)
- `heCEntLines`: `heDEntID` (FK to `heDocEntries`), `heItemDescription` (display override)
- `heDocEntries`: `heID, heOfficialDateTime, heSourceType` (=9 means imported via API/scenario)

## UX Flow

Implemented steps marked ✅; pending steps marked ⏳.

1. ✅ User opens the web app on phone. (Cloudflare Access in front: pending.)
2. ✅ Taps microphone button.
3. ✅ Speaks Greek command like *"Βάλε στον Καραμήτρο 3 ώρες αλλαγή φόρμας..."*.
4. ✅ Web Speech API converts to text. Interim results render live; textarea remains editable.
5. ⏳ NLP parser extracts: customer name, quantity, unit, description. (Today the whole transcript is sent as the search term.)
6. ✅ User taps "Αναζήτηση" — explicit confirmation, no auto-search. The whole transcript is sent to `/api/customers/search`, which calls `CustomerSearch` with progressive shortening.
7. ✅ Tappable result list with caption "Βρέθηκαν N πελάτες για «<matched>»".
   - Today every result count is shown the same way. Future: 1-match auto-select, 2-5 tappable list, 6+ top 5 + text filter, 0 → "Δεν βρέθηκαν πελάτες" (already implemented for the 0 case).
8. ✅ User taps a row → selected-customer card with "Αλλαγή".
9. ⏳ Confirmation screen with all parsed details + customer + Καταχώρηση ✓.
10. ⏳ Call `SalesOrders` via Pylon API on submit.
11. ⏳ Success → show document ID. Failure → show error with retry.
12. ⏳ Every entry logged to the local SQLite audit DB.

## Configuration (config-driven)

Everything that varies per deployment goes in `config.json` mounted from the host:

```json
{
  "pylon": {
    "apiUrl": "http://192.168.200.100:7024/exesjson",
    "apiCode": "...",
    "applicationName": "Hercules.MyPylonCommercial",
    "databaseAlias": "...",
    "username": "...",
    "password": "..."
  },
  "defaults": {
    "seriesCode": "ΠΑΡ",
    "csbrName": "Έδρα",
    "pmmtCode": "...",
    "defaultItemCode": "..."
  },
  "search": {
    "customerSearchScript": "CustomerSearch"
  },
  "ui": {
    "language": "el",
    "speechLang": "el-GR",
    "confirmBeforeSubmit": true
  },
  "parser": {
    "mode": "regex"
  }
}
```

`config/config.example.json` ships placeholders. `config/config.json` is git-ignored. Loader reads `CONFIG_PATH` env var (default `./config/config.json`) and fails fast on missing required fields.

`npm run setup` (CLI wizard) and `npm run test-config` (diagnostic) are wired up; `setup` is still a stub, `test-config` runs the full elogin → getdata flow against a real server.

## Error Handling Strategy

Fail-fast. If a Pylon API call fails, show a clear error to the user with a retry button. No silent retries (except the one-shot re-login on session-expiry-shaped errors). No local queues. All errors logged to the audit DB once that exists.

The Pylon client validates the **outer envelope** first (`Status === "OK"`) before unwrapping `Result`. This is the lesson learned from the CustomerSearch "Package size > max" bug — earlier the client only checked the inner payload, so server-side errors silently returned `{}`.

## Audit Log (SQLite) — PENDING

Schema:

```sql
CREATE TABLE entries (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  created_at DATETIME NOT NULL,
  user_email TEXT NOT NULL,
  raw_transcript TEXT,
  parsed_customer TEXT,
  parsed_quantity REAL,
  parsed_description TEXT,
  matched_customer_id TEXT,
  matched_customer_name TEXT,
  status TEXT,
  pylon_doc_id TEXT,
  error_message TEXT,
  duration_ms INTEGER
);
```

Provide a "History" view in the UI showing last 50 entries per user. (Pending.)

## Deployment

- **Dev:** codebox Linux VM (where you're running now) — Pylon API at `192.168.200.100:7024`. Dev server on whatever port is free (`startpcs` typically holds 3000 → fallback to 3001).
- **Production for Bozas:** pylord VM at `192.168.200.7` (Ubuntu 22.04, 2 GB RAM, 2 vCPU). Not yet deployed.
- **For SSH from codebox to pylord:** user is `stab`.
- **For future customers:** Synology Container Manager OR Ubuntu VM on their infrastructure.

### Development server cross-origin (Cloudflare Tunnel)

Next.js 16 blocks cross-origin requests to dev-only assets/endpoints by default (HMR was returning 401 over a Cloudflare Tunnel). The supported escape hatch is `allowedDevOrigins` — **top-level** in v16, NOT under `experimental`. `next.config.ts` ships:

```ts
allowedDevOrigins: ["*.trycloudflare.com", "*.cloudflare.com"]
```

When we move to a permanent tunnel with a custom domain, add that hostname to the same array. Verified locally that `Origin: https://*.trycloudflare.com` passes the cross-site dev block while arbitrary origins still return 403.

## Critical rules

1. **Do not guess Pylon database column or table names.** Ask the owner via `questions.md` before assuming anything.
2. **Do not invent Pylon API behaviors.** All Pylon-specific knowledge comes from the owner or from official scripts already in the template. When unsure, write a probe script under `scripts/` and ask the owner to run it.
3. **All UI text in Greek.** All code, comments, and commit messages in English.
4. **Config-driven from day one.** No hardcoded values that would prevent deployment to another customer.
5. **Mobile-first.** Always design and test for phone screens first, then desktop.
6. **Never commit secrets.** API codes, passwords, customer data → never in git. Only in mounted `config.json`.
7. **Outer envelope first.** Always validate the Pylon outer envelope (`Status`, `Error`) before unwrapping `Result`. Inner-only checks miss server errors. (The client does this — don't bypass it by calling fetch directly.)

## Pending tasks

In rough order of dependency:

1. **NLP parser (regex strategy)** — `src/lib/parser/`. Extract `{ customerName, quantity, unit, description }` from the transcript. Pluggable behind a strategy interface so the LLM-based parser can drop in later. Pipe `customerName` (not the whole transcript) into `/api/customers/search`.

2. **SalesOrders write path** — three pieces:
   - Pylon-side modifications to the `SalesOrders` script (see "Pylon API Scripts we use #2").
   - `src/app/api/orders/route.ts` server-side POST endpoint that calls `postData("SalesOrders", { CstmID, SeriesCode, CsbrName, PmmtCode, Lines })`.
   - `Καταχώρηση ✓` submit button in `QuickEntry`, success-state UI (document id), retry-on-failure.

3. **Audit log (SQLite)** — `src/lib/db/` with `better-sqlite3`, schema above, write-on-submit + History view in the UI showing the last 50 entries per user.

4. **Cloudflare Access integration** — read `Cf-Access-Authenticated-User-Email` server-side and stamp the audit log with `user_email`. No custom login.

5. **Cloudflare Tunnel for production** — proper named tunnel with a custom domain (not `*.trycloudflare.com`); add the production hostname to `allowedDevOrigins` for any dev-server hitchhiking.

6. **Docker packaging** — multi-arch if feasible, config mounted from host, ready for Synology Container Manager or Ubuntu VM deployment.

7. **CLI setup wizard (`npm run setup`)** — interactive `config.json` builder with a live Pylon connection test before saving. Currently a stub.

8. **UI polish for match-count heuristics** — 1 match → auto-select with confirmation; 2-5 → tap list (today); 6+ → top 5 + text filter; 0 → friendlier prompt with re-speak / type fallback.

## Communication with claude.ai

- `current-task.md` tells you what to work on right now.
- If you have questions, write them to `questions.md`, commit and push. The owner will tell claude.ai to read them.
- claude.ai responds via updates to `current-task.md` or new revisions of `questions.md`.

## Repository

- **Source code (private):** `github.com/stabmegane/bozas-quick-entry`
- **This bridge (public):** `github.com/stabmegane/bozas-quick-entry-bridge`
