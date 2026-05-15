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

| Layer | Choice |
|-------|--------|
| Framework | Next.js 14+ with App Router |
| Language | TypeScript |
| UI | Tailwind CSS |
| Local DB | SQLite (better-sqlite3) for audit log |
| HTTP | Native fetch |
| Voice | Web Speech API (browser-native, free, Greek support) |
| NLP | Strategy pattern: RegexParser first, LLMParser pluggable later |
| Container | Docker (multi-arch if feasible) |
| Remote access | Cloudflare Tunnel |
| Authentication | Cloudflare Access (no custom login) — reads Cf-Access-Authenticated-User-Email header |

## Pylon Integration

### Connectivity API basics

- Base URL: http://<pylon-server>:7024/exesjson/
- Endpoints: elogin, getdata, postdata
- Auth: session cookie returned by elogin, used in subsequent calls
- All payloads in JSON

### Credentials

These come from the Pylon API Template. The current dev template:
- **API Code:** UOCUQZNWVDBG7OA
- **Username/Password:** to be provided via config.json
- **Application name:** Hercules.MyPylonCommercial (confirm with owner)
- **Database alias:** to be provided

### Pylon API Scripts we use

**1. Customer Search (NEW — needs to be created in Pylon API Template)**

A new Get-type script CustomerSearch that:
- Accepts @search parameter (raw text, Greek or Latin)
- Calls sc$String.SoundexConvertSqlLike(searchText) to convert to Pylon's soundex
- Executes SQL query against HECUSTOMERS joined with HETRADERS and HECONTACTS
- Filters by HECOMPID = @$System$CurCompany and HEACTIVE = 1
- Searches LIKE '%soundex%' on HENAMESOUNDEX column
- Returns: CSTMID, CSTMCODE, CSTMNAME, CSTMTIN, TRDRID

**2. SalesOrders (EXISTS — may need modifications)**

The existing SalesOrders script in template UOCUQZNWVDBG7OA:
- Accepts payload with CstmID, SeriesCode, CsbrName, PmmtCode, Lines[]
- Creates a Sales Order entry (CmdtType = 2, SourceType = 9)
- Returns { EntityId, CustomerId, Messages }

**Modifications needed:**
- Add reading of CenlComments per line → line.SetValue("Comment", ...)
- Add reading of CenlItemDescription per line → for display override on printouts
- Change behavior: do NOT create new customer if not found; return error instead

### Database structure (DO NOT GUESS column names — ask owner)

Known so far:
- heCustomers: heID, heCode, heName, heNameSoundex, heTrdrID, heCompID, heActive
- heTraders: heID, heCntcID
- heContacts: heID, heTIN
- heTraderBranches: heID, heTrdrID, heName (most have one row for "Έδρα", some have more)
- heCEntLines: heDEntID (FK to heDocEntries), heItemDescription (display override)
- heDocEntries: heID, heOfficialDateTime, heSourceType (=9 means imported via API/scenario)

## UX Flow

1. User opens web app on phone (already authenticated via Cloudflare Access)
2. Taps microphone button
3. Speaks Greek command like *"Βάλε στον Καραμήτρο 3 ώρες αλλαγή φόρμας..."*
4. Web Speech API converts to text (with manual edit fallback)
5. NLP parser extracts: customer name, quantity, unit, description
6. App calls CustomerSearch via Pylon API with the customer name
7. Soundex search returns matches
8. UI behavior:
   - 1 match → auto-select, show confirmation
   - 2-5 matches → tap-able list
   - 6+ matches → top 5 + text search
   - 0 matches → "not found, speak again or type"
9. Confirmation screen shows all parsed details + customer
10. User taps "Καταχώρηση ✓"
11. App calls SalesOrders via Pylon API
12. Success → show document ID. Failure → show error.
13. All entries logged to local SQLite audit DB

## Configuration (config-driven)

Everything that varies per deployment goes in config.json mounted from the host:

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

Provide a CLI setup wizard (npm run setup) that interactively builds this config with validation (test connection to Pylon API before saving).

Provide a npm run test-config diagnostic command.

## Error Handling Strategy

Fail-fast. If Pylon API call fails, show clear error to the user with a retry button. No silent retries, no local queues. All errors logged to audit DB.

## Audit Log (SQLite)

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

Provide a "History" view in the UI showing last 50 entries per user.

## Deployment

- **Dev:** codebox Linux VM (where you're running now) — Pylon API at 192.168.200.100:7024
- **Production for Bozas:** pylord VM at 192.168.200.7 (Ubuntu 22.04, 2GB RAM, 2 vCPU)
- **For SSH from codebox to pylord:** user is stab
- **For future customers:** Synology Container Manager OR Ubuntu VM on their infrastructure

## Critical rules

1. **Do not guess Pylon database column or table names.** Ask the owner via questions.md before assuming anything.
2. **Do not invent Pylon API behaviors.** All Pylon-specific knowledge comes from the owner or from official scripts already in the template.
3. **All UI text in Greek.** All code, comments, and commit messages in English.
4. **Config-driven from day one.** No hardcoded values that would prevent deployment to another customer.
5. **Mobile-first.** Always design and test for phone screens first, then desktop.
6. **Never commit secrets.** API codes, passwords, customer data → never in git. Only in mounted config.json.

## Communication with claude.ai

- current-task.md tells you what to work on right now.
- If you have questions, write them to questions.md, commit and push. The owner will tell claude.ai to read them.
- claude.ai responds via updates to current-task.md or new briefings.

## Repository

- **Source code (private):** github.com/stabmegane/bozas-quick-entry
- **This bridge (public):** github.com/stabmegane/bozas-quick-entry-bridge
