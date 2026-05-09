---
name: api-contract-validation
description: >
  Validates that API response shapes match frontend expectations during code review.
  Use this skill whenever you are reviewing or merging a PR that touches API route handlers,
  API client files, error-handling paths, or SSE/streaming event shapes. Also invoke when
  you're implementing a backend change that returns data to the frontend, adding or renaming
  response fields, changing error objects, or refactoring a fetch utility. This skill catches
  cross-layer contract drift — the class of bug where backend and frontend silently disagree
  on field names, types, error shapes, or streaming event structure — before it reaches
  production. Prior incidents (TEC-1578: plain-object errors broke all frontend error handling;
  TEC-1372: stale test assertions missed a shape change; TEC-1559: SSE field rename broke RAG
  streaming) all share this root cause. If the diff touches anything between a server response
  and a frontend consumer, invoke this skill even if the change looks trivial.
---

# API Contract Validation

The goal is simple: make sure the server and every client that calls it agree on what the
response looks like — fields, types, error shapes, and (for streaming) event structure.
This review step catches drift before it hits production.

---

## Step 1 — Identify the contract surface

Scan the diff for these change categories. List each one you find.

| Category | What to look for |
|---|---|
| Route handler changes | Added, removed, or renamed fields in `res.json(...)` / `return { ... }` |
| Error shape changes | `throw new Error(...)` vs `throw { message, code }` vs plain objects; `next(err)` shape |
| API client changes | `fetch`/`axios` wrappers, response destructuring in `api.js`, `apiClient.js`, etc. |
| Frontend consumer changes | Components, hooks, or stores that destructure or access response fields |
| SSE / streaming changes | `res.write(...)`, `event: ...`, `data: ...` field names in server-sent events |
| Type / interface changes | TypeScript interface or Zod schema edits on either side of the API boundary |

If the diff touches none of these, this skill is not needed — close it out.

---

## Step 2 — Trace each affected contract

For every item found in Step 1, answer:

**On the server side:**
- What fields does the endpoint return? List them with their types.
- What does the error path return? Is it an `Error` instance, a plain object, or a string?
- For SSE: what event names and field names are emitted?

**On the client side:**
- What fields does the frontend code access or destructure from the response?
- How does error-handling code inspect the error — `.message`, `.code`, `instanceof Error`, `error.response.data`?
- For SSE: what event names and field names does the client listen for?

Write this out explicitly. The contract mismatch is often invisible until it's written down side by side.

---

## Step 3 — Flag mismatches

Compare Step 2's server and client inventories. Call out any of these:

| Mismatch type | Example |
|---|---|
| Missing field | Server returns `{ userId }`, client reads `.id` |
| Renamed field | Server returns `label`, client reads `name` |
| Type change | Server returns `count` as string, client parses it as `Number` |
| Error shape drift | Server throws plain `{ message }` object, client does `err instanceof Error` |
| SSE field rename | Server emits `event: chunk` with `data: { text }`, client listens for `content` |
| Null/undefined unsafety | Server may omit an optional field, client accesses it unconditionally |
| Stale test assertion | Test asserts the old shape while the implementation returns a new one |

For each mismatch, state:
- Where it originates (file + line if possible)
- What breaks at runtime (silent undefined, thrown exception, broken render)
- Severity: **blocking** (will break at runtime) or **advisory** (edge case, degraded UX)

---

## Step 4 — Check test coverage alignment

After a shape change, tests often lag behind. Check:

1. Are there existing contract tests, fixture tests, or snapshot tests that assert response shape?
2. If yes, do they still match the new shape? A test asserting the old field name will silently
   pass if the assertion is not strict (`toHaveProperty` vs `toEqual`).
3. Are there no tests? Flag this as advisory if the endpoint is non-trivial.

Stale assertions are especially dangerous — they give false confidence that the shape is
correct when the test no longer exercises the current code path.

---

## Step 5 — Validation checklist

Paste this into your review comment. All items must be resolved before approving.

```
## API Contract Validation

**Contracts reviewed:**
- [ ] [Endpoint or stream name — e.g. "GET /api/widgets"] — [server file] → [client file]

**Shape comparison:**
| Field | Server returns | Client expects | Status |
|---|---|---|---|
| `fieldName` | `string` | `string` | OK |
| `deletedField` | — | `string` | MISMATCH |

**Error handling:**
- [ ] Error shape is consistent (`Error` instance vs plain object vs response body)
- [ ] Client error handler matches what the server actually throws or rejects with

**SSE / streaming (if applicable):**
- [ ] Event names match
- [ ] Data field names match

**Test coverage:**
- [ ] Existing shape assertions updated to match new fields
- [ ] No test asserting the old (now-wrong) shape

**Result:** PASS / ADVISORY / BLOCK
```

### Result key

| Result | Meaning |
|---|---|
| `PASS` | No mismatches found. |
| `ADVISORY` | Low-severity drift or missing tests — does not block merge, but should be tracked. |
| `BLOCK` | Runtime-breaking mismatch. Do not merge until resolved. |

---

## Common patterns and how to resolve them

### Pattern: error shape drift

The most common cause of silent frontend failures. A route handler doing:

```js
// Bad — plain object, not an Error instance
next({ message: 'Not found', status: 404 });
```

…while client code does:

```js
// Assumes Error instance
catch (err) { console.error(err.message); }  // works, lucky
toast(err instanceof Error ? err.message : 'Unknown error'); // branch never taken
```

Resolution: standardise on either `next(new Error(...))` with `err.status` attached, or a
documented plain-object shape — and update every handler and every client to match.

### Pattern: SSE field rename

```js
// Server emits
res.write(`data: ${JSON.stringify({ content: chunk })}\n\n`);

// Client listens for
source.addEventListener('message', e => {
  const { text } = JSON.parse(e.data); // 'text' is undefined — silent empty render
});
```

Resolution: rename one side, update tests, and add a comment at the `res.write` call and
the `addEventListener` call naming the shared contract so future reviewers see the coupling.

### Pattern: stale fixture / snapshot

```js
// Test from 6 months ago
expect(res.body[0]).toHaveProperty('name'); // passes even after 'name' → 'label' rename
                                             // because toHaveProperty returns false, not throws
```

Use `toEqual(expect.objectContaining({ label: expect.any(String) }))` and re-run the fixture
capture after any shape change to keep the assertion authoritative.

---

## Relationship to other skills

| Skill | Phase | Focus |
|---|---|---|
| `api-contract-validation` (this skill) | Code review | Cross-layer shape alignment — does the frontend expect what the backend returns? |
| `payload-shape-gate` | Implementation (pre-merge) | G2 gate — is there a contract test, captured fixture, or Playwright smoke for each fetch flow? |
| `code-review` | Code review | General quality, correctness, style |

These skills are complementary. `payload-shape-gate` asks *"is there a test?"*;
this skill asks *"do the two sides of the API agree?"* Both checks are needed.

---

*TEC Custom Skill — maintained by the Deltek Technical Services Engineering team.*
