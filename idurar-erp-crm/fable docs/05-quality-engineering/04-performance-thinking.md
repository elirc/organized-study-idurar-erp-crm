# Performance Thinking

The rule first: **measure before optimizing.** This file names likely hotspots with anchors, but every one is a *hypothesis* until profiled. Guessing at performance is how you spend a week speeding up something that was never slow.

## Performance domains in this app

| Domain | Where it lives | Likely hotspot | How to measure |
| --- | --- | --- | --- |
| DB queries | Mongoose calls | missing indexes, N+1 via autopopulate | `explain()`, DB profiler, query logging |
| Server CPU | request handlers | PDF generation, ReDoS regex | Node `--prof`, event-loop lag metric |
| Network | axios ↔ API | render-blocking settings, no caching | DevTools Network tab |
| Render | React components | large tables, unmemoized selectors | React DevTools Profiler |
| Bundle | Vite build | lazy-loading gaps | `vite build` + bundle visualizer |
| Memory | long-running | unbounded `loggedSessions`, PDF temp files | heap snapshots |

## Likely hotspots, ranked (all hypotheses until measured)

### 1. PDF generation blocks the event loop
`pdfController/index.js:70-79` renders synchronously with `html-pdf` (PhantomJS) on the request thread, and regenerates on every download with no cache (`:25-27`). On a single-threaded runtime, one PDF render stalls every concurrent request. **Fix:** move to a worker thread or a queue; cache the rendered PDF in S3 (already a dependency). **Measure first:** event-loop lag during a download, and download frequency.

### 2. Hidden N+1 via global autopopulate
`Payment` autopopulates `invoice`, which autopopulates `client` (`Payment.js:20-25`, `Invoice.js:31-36,178`). Listing 100 payments can trigger a cascade of joins. **Fix:** populate explicitly per query (like `read.js:11`) rather than globally. **Measure first:** enable Mongoose debug logging (`mongoose.set('debug', true)`) and count queries per list request.

### 3. Missing indexes on hot paths
Login email lookup (`login.js:31`), invoice list filter (`paginatedList.js:32`) — no declared indexes means collection scans as data grows. **Fix:** add `Admin.email`, `{Invoice.removed, client}` indexes. **Measure first:** `explain('executionStats')` — look for `COLLSCAN`.

### 4. Offset pagination on deep pages
`skip((page-1)*limit)` (`paginatedList.js:4,37`) scans and discards skipped docs. Page 1000 scans 10,000 rows. **Fix:** cursor/keyset pagination for large collections. **Measure first:** query time at high page numbers.

### 5. `countDocuments` on every list request
`paginatedList.js:44` runs a full count alongside every list. **Fix:** `estimatedDocumentCount()` for large collections, or cache the count. **Measure first:** is the count query the slow half of the `Promise.all`?

### 6. Render-blocking settings fetch
`ErpApp.jsx:36-42` blocks the entire app shell on the settings request. A slow settings call = whole-app stall. **Fix:** render with placeholders, hydrate settings async. **Measure first:** settings endpoint latency; time-to-interactive.

### 7. Unmemoized / factory selectors
`selectItemById` (`selectors.js:8-9`) creates a new selector per call; used inline it defeats reselect memoization, causing extra recomputes. **Measure first:** React Profiler flamegraph on list interactions.

## How to find each class of problem

- **N+1:** turn on `mongoose.set('debug', true)`; if one API call logs dozens of queries, you found it.
- **Missing index:** `db.invoices.find({...}).explain('executionStats')` → `COLLSCAN` + high `docsExamined`.
- **Slow render:** React DevTools Profiler → components that render often or long.
- **Big bundle:** `vite build` then inspect chunk sizes; `rollup-plugin-visualizer`.
- **Event-loop block:** measure lag with a periodic timer; a spike during PDF/search confirms CPU blocking.

## The senior habit

Every performance claim in this file is labeled a hypothesis on purpose. In an interview, "I'd profile first with `explain`/the React Profiler, then optimize the actual hotspot" beats "I'd add indexes and memoize everything." Premature optimization adds complexity for wins that may not exist. Optimize the measured bottleneck, then measure again.

## Interview angle

"How do you find and fix a performance problem?" → measure (name the tool per domain) → find the bottleneck → fix root cause → re-measure. Use the N+1 (autopopulate) and the PDF event-loop block as your concrete examples. See [08-interview-prep/03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md) Q7/Q9/Q11.

## Drill

Pick hotspot #2. Write the exact command to turn on Mongoose query logging, predict how many queries a `GET /api/payment/list?items=10` would emit given the autopopulate chain, then state the explicit-populate fix. Self-grade — Strong: you reason about the cascade depth (payment → invoice → client) and propose per-query populate, not "remove autopopulate everywhere."
