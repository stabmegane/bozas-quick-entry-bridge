# Questions from Claude Code

> Use this file to ask questions when something is ambiguous or you need clarification. Commit and push. The owner will tell claude.ai to read them, and answers will appear in `current-task.md` or as a new revision of this file.

## Format

Each question goes under a dated heading. Mark it `status: open` when asked, `status: answered` when resolved.

Example:

```
### 2026-05-13 — Pylon cookie expiry handling

status: open

When the Pylon API cookie expires mid-session, should the web app:
(a) Auto-re-login transparently and retry the original request
(b) Fail with a clear error and ask the user to refresh

The BRIEFING says "fail-fast" but also mentions "auto re-login once" in the error handling section. Which is it?
```

---

## Open Questions

### 2026-05-15 — How does Pylon signal cookie expiry?

status: open

The task spec says: *"Auto re-login ONCE on cookie expiry, then retry the original request."* That requires the client to distinguish an expired-cookie response from any other server-side error, so we know when to re-login and when to fail.

Pylon docs don't seem to pin down the exact signal. The client currently treats a response as session-expired (and re-logs in once) when **any of** the following is true:
- HTTP status is 401 or 403, **or**
- the error message / body / `Messages` payload contains (case-insensitive) `cookie`, `session`, `expired`, `not logged`, `unauthor`, or both `login` + `again`.

If a real expired-cookie response looks different (e.g. `Status: "EXPIRED"`, `Error.Code: 1019`, a specific `Messages[0].Text` string, etc.), please share an example so we can tighten the check. Otherwise we'll keep the heuristic and refine it the first time we see expiry in the wild.

### 2026-05-15 — Pylon response envelope for `getdata`/`postdata`

status: open

The briefing confirms `elogin` uses the double-encoded envelope (`{ "Result": "<json-encoded string>" }`). It's not explicit about `getdata` / `postdata`:
- Do those responses use the same `Result` envelope, or do they return the payload directly?
- For `getdata`, are the rows under `rows`, `Rows`, `Data`, or something else? Is there a separate `TotalRows` / `HasMore` field for pagination?

The client currently *unwraps* a `Result` string if present (otherwise returns the body as-is), and the diagnostic searches a handful of likely key names (`rows`, `Rows`, `Data`, `data`, `Result`, `Results`, `Items`, `items`) to find the array. Once you run the diagnostic we'll see what shape comes back and can lock the types down.

## Answered Questions

(none yet)
