# Current Task

> This file describes what Claude Code should be working on right now. It is overwritten by claude.ai as work progresses.

## Status: Not started

## Task: Pylon API client + diagnostic

Implement the client that talks to the Pylon Connectivity API. No UI, no voice, no parser. We just want a robust, typed client and a diagnostic command that proves it works against a real Pylon server.

### Goals

1. **Pylon client** in `src/lib/pylon/` that exposes:
   - `login()`: calls `elogin`, returns and internally caches the cookie
   - `getData(entityCode, options?)`: wraps `getdata` with the cached cookie
   - `postData(entityCode, data)`: wraps `postdata` with the cached cookie
   - Auto re-login ONCE on cookie expiry, then retry the original request. If it fails again, throw.
   - Typed requests and responses based on the shapes in the Pylon docs and the actual `SalesOrders` script in the API template.

2. **Types** in `src/lib/pylon/types.ts`:
   - `PylonLoginRequest`, `PylonLoginResponse`
   - `PylonGetDataRequest`, `PylonGetDataResponse`
   - `PylonPostDataRequest`, `PylonPostDataResponse`
   - `PylonError` class that preserves the server's `Messages` payload if present

3. **Error handling**:
   - Network errors (fetch rejecting, timeout) → throw `PylonError` with clear message
   - HTTP non-2xx → throw `PylonError` including status + body
   - Pylon returned a body with `Status != "OK"` or with an `Error` field → throw `PylonError` with that information
   - No silent failures

4. **Cookie cache**: in-process singleton (store in module state). No disk persistence. Cookie is re-fetched on the first call after process start and re-fetched once if expired.

5. **Diagnostic script** `scripts/test-config.ts`:
   - Run with `npm run test-config`
   - Loads config
   - Performs `elogin` and reports `Cookie OK` or error
   - Calls an existing Get script from the API template to prove data flow works. Try `Items` with `packagenumber: 1` and `packagesize: 1`. Report how many rows came back and the first row's keys.
   - Coloured, readable output with ✅ / ❌ next to each step
   - Exit code 0 on full success, 1 on any failure

6. **Add `npm run test-config`** script in package.json.

7. **Unit tests are not required yet.** The diagnostic against the real server is our integration test.

### Config

The owner will provide a real `config/config.json` with test credentials before running the diagnostic. Assume the following placeholder shape is correct and don't invent extra fields:

```json
{
  "pylon": {
    "apiUrl": "http://192.168.200.100:7024/exesjson",
    "apiCode": "UOCUQZNWVDBG7OA",
    "applicationName": "Hercules.MyPylonCommercial",
    "databaseAlias": "…",
    "username": "…",
    "password": "…"
  },
  ...
}
```

If the existing `config.example.json` misses any field that the real Pylon call needs, update the example to include it. Keep placeholders (no real secrets).

### Request shapes (from samples)

**Login** (POST `{apiUrl}/elogin`):
```json
{
  "username": "...",
  "password": "...",
  "apicode": "...",
  "applicationname": "Hercules.MyPylonCommercial",
  "databasealias": "..."
}
```

The response body has a `Result` field that contains a JSON-serialized string with the actual payload inside. The `cookie` is nested in that inner JSON. Parse it carefully. See the Postman collection example in the project's Pylon docs for the double-encoding pattern.

**Getdata** (POST `{apiUrl}/getdata`):
```json
{
  "cookie": "...",
  "apicode": "...",
  "entitycode": "Items",
  "packagenumber": 1,
  "packagesize": 2000,
  "extras": "{\"@itemIDs\":[...]}"
}
```

`extras` is stringified JSON when used; the client should accept a plain object from the caller and stringify it internally.

**Postdata** (POST `{apiUrl}/postdata`):
```json
{
  "cookie": "...",
  "apicode": "...",
  "entitycode": "SalesOrders",
  "data": "{\"CstmID\":\"...\",\"SeriesCode\":\"ΠΑΡ\",\"Lines\":[...]}"
}
```

Same pattern: `data` is a stringified JSON. The client accepts an object and stringifies it.

### Constraints

- No UI, no routes, no react components in this task. Server-side / library code only.
- No hardcoded URLs or credentials — everything comes from config.
- No external HTTP library; use native `fetch`.
- Fail-fast. No silent retries except the one-time re-login.
- If anything is ambiguous (e.g. response shape details don't match reality), stop and add a question to `questions.md` in the bridge repo.

### Acceptance criteria

- `npm run test-config` run from the codebox connects to the real Pylon at 192.168.200.100:7024 using the provided config, prints ✅ for login and getdata, and exits 0.
- Code is typed, compiles cleanly (throu --noEmit), and passes ESLint.
- config/config.example.json remains up-to-date.

### When done

Update this file's Status to `Completed` and add a short summary. Then ask the owner what to work on next.
