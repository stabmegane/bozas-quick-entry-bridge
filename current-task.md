# Current Task

> This file describes what Claude Code should be working on right now. It is overwritten by claude.ai as work progresses.

## Status: Not started

## Task: Pylon response probing

The first diagnostic run succeeded for elogin and getdata. But the client reported "0 rows" because it couldn't find the rows array inside the response's Data field. The top-level keys were Cookie, Data, Messages.

The Pylon Items script packs multiple DataTables into the response via:

  dic["Data"] = sc$APIContext.BuildResultSet(tblItems, tblPhotosByItmIDs, tblAttrsByItmIDs);

So Data is a resultset containing multiple named tables. We need to see the exact shape before we can type it correctly.

### Goals

1. Create scripts/probe-getdata.ts that:
   - Loads config
   - Logs in via the Pylon client
   - Calls getData("Items", { packagenumber: 1, packagesize: 2 })
   - Pretty-prints the full raw response with JSON.stringify(resp, null, 2)
   - No colour, no fancy formatting - just the raw JSON dumped to stdout
   - On failure, prints the error and exits non-zero

2. Add npm run probe-getdata script in package.json.

3. Do not touch the client's response parsing yet. We will iterate on the client after we see the real shape.

4. Do not push the probe's output to GitHub. Only the script code goes to git.

### Constraints

- Same as before: work only in ~/bozas-quick-entry/, fail-fast, no hardcoded values.
- The goal is information gathering, not feature completion. We just want to see the raw structure.

### When done

Update this file's Status to Completed and add a summary. Tell the owner to run npm run probe-getdata.
