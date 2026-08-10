---
name: call-stack
description: Print a runtime call stack in the terminal — the whole repo, a feature, or a tagged file, traced call by call down to its leaves, with what the trace exposes.
disable-model-invocation: true
---

# Call stack

Trace the codebase's **runtime** paths and print them as ASCII trees in the terminal. A call stack, not an architecture diagram: every line is a call that actually happens when the code runs, nested under its caller, in the order it executes.

## 1. Pick the roots

A **root** is a node a tree starts from. `$ARGUMENTS` decides which roots:

- **A file or symbol tagged** — `@src/queue/worker.ts`, `OrderProcessor.run`, `src/api/checkout.ts:handleCheckout`. Roots are that file's public surface: every exported function, class method, route handler, or command defined in it. A tagged symbol is the single root.
- **A directory or feature named** — roots are the entry points reachable inside it.
- **Empty** — roots are every entry point in the repo.

An **entry point** is where execution starts from outside the process. Sweep the scope for each kind that exists:

- HTTP route handlers, RPC/GraphQL resolvers, webhook receivers
- CLI commands, `main`, scripts in `package.json` / `Makefile` / `pyproject.toml`
- scheduled work — cron config, `vercel.json` crons, timers
- queue and stream consumers, event/message listeners
- client-side handlers that cross the network (form submit, mutation hooks)
- exported public API of a library

A root reached from inside the process — the common case for a tagged file — is an orphan until you say who calls it. Grep the repo for each caller and render them as a `REACHED FROM` list above its tree, one line per calling site, walked back to the entry point that starts it.

**Enumerate the whole roster before tracing any of it.** Finding a root is a grep; tracing one is an afternoon. Take the cheap pass first and write the full list down — every root in scope, none traced yet.

The roster is what makes an incomplete trace honest. Decide what to trace by subtracting from a list you have already written, and whatever you skip is on the page by construction. Decide as you go and the roots you never noticed are the ones the reader never hears about — an output that looks complete because its gaps are invisible.

Then budget. Roster of eight or fewer: trace all of them. Larger: trace the top eight, ranked by how directly the root faces a user —

1. synchronous user-facing — request handlers, form submits, public API
2. asynchronous user-facing — queue consumers and jobs that finish a user's work
3. scheduled — cron, sweeps, reconciliation
4. internal — admin routes, migrations, dev tooling

— and print every remaining root by name under `NOT TRACED`, with the rank that put it there. Say the count: `NOT TRACED — 23 roots`. A reader who disagrees with the cut re-runs the skill scoped to the one they wanted.

Done when: every root in scope is on the roster as a `file:symbol` pair with what triggers it or which callers reach it, and each is marked for tracing or for `NOT TRACED`.

## 2. Trace each root to its leaves

Open the root's body. Take each call it makes in source order, open the callee, repeat. Depth is set by leaves, not by a level count.

A branch stops at a **leaf**:

- **vendor boundary** — third-party API, database, cache, blob store, mail provider, model provider
- **service seam** — a call that continues in code you own but this repo does not hold: internal HTTP or RPC, a queue another repo consumes, an event another team listens to. A true leaf here is a lie — the tree continues elsewhere. Mark it `↗` and name the far side.
- **terminal outcome** — an HTTP status returned, a value returned to the caller, an error thrown
- **already drawn** — a subtree rendered under another section; name the node and point at that section instead of redrawing it
- **dependency internals** — stop at the library call you make, e.g. `Cheerio extraction`

Rules that keep the tree true to the code:

- Every rendered name is a symbol you read in a file, or an external service that symbol calls.
- Siblings sit in execution order.
- A conditional renders as a question node with its outcomes beneath it: `outcome already persisted?` → `└─ YES → …`.
- Concurrency renders as its combinator — `Promise.all()`, `asyncio.gather()`, `errgroup` — with the concurrent calls as children.
- Retry, timeout, and failure paths are calls too. Render them.

Mark what the call name alone cannot tell the reader. Four markers, each carrying something no reader could guess from a symbol — how often it runs, whether anyone waits for it, where it goes, and what it changes:

- `× N` — the call runs inside a loop or a `map`. Name what it iterates: `× per line item`. One rendered node standing for fifty calls is the difference between a diagram and a fan-out you can see.
- `⇢` — the call is dispatched without being awaited: fire-and-forget, floating promise, background goroutine, `after()`. Its failures do not reach the caller.
- `↗` — the call leaves for another service or repo you own. Name the far side: `↗ billing-svc`. This is where the tree ends and someone else's begins.
- `!` — the call mutates durable state. Name the resource: `! writes orders`, `! deletes holds`, `! publishes order.created`. Reads and writes render identically without it, so the tree answers "what calls what" and never "what changes what".

Annotate with the rest of what you read in the code and the name does not carry: schedules, timeouts, retry counts, status codes — `Cron — every 5 minutes`, `60-second timeout`, `HTTP 201 / 409 / 422`. Facts only, read from a file. A marker you cannot point at a line for does not go on the tree.

**Budget the depth.** A subtree can be larger than the trace is worth. When one is, render its first level and close it with `… deeper not traced`, then move on. A branch honestly cut off is worth more than a branch quietly flattened, because the reader can see where to look next.

Done when: every root has a tree whose every branch ends on a leaf or a declared cut, every node came from a file you opened, and every looped, unawaited, outbound, or mutating call carries its marker.

## 3. Render

One numbered section per root, in trigger order. A subtree reached by two or more roots gets its own numbered section, and each caller points at it — that is what keeps the trees short.

Format contract:

```text
TITLE IN CAPS — RUNTIME CALL STACK
==================================

1. SECTION NAME
---------------

rootCall()
├─ siblingCall()
│  └─ nestedCall()
└─ lastCall()
   └─ External Service
```

A root with callers carries them above its tree:

```text
2. WORKER ENTRY
---------------

REACHED FROM
  POST /api/checkout → enqueue() → OrderProcessor.run()
  Cron every 5 min → handleSweep() → enqueue() → OrderProcessor.run()

OrderProcessor.run(orderId)
├─ …
```

- Title names the traced surface — the repo, the feature, or the tagged file's path — underlined with `=`. Section headings underlined with `-`. Both match the text length.
- Glyphs: `├─` for a sibling with more below, `└─` for the last, `│  ` to carry a line down, three spaces where a `└─` has closed.
- Blank line between logical groups inside a long section.
- Roots left on the roster get a final `NOT TRACED` section, counted, one line each.
- Close the whole output with the primary dependency chain — layers from entry point down to infrastructure, joined by `→`, with any `↗` seam marked as the separate repo it is.

Then close with a **source index** — the evidence trail. The tree stays clean; the index makes it checkable:

```text
SOURCE INDEX
------------

1. CHECKOUT SUBMISSION   src/api/checkout.ts:14
   Cart.load              src/cart/store.ts:88
   Inventory.reserve      src/inventory/reserve.ts:31
```

One line per symbol you rendered, in tree order, pointing at the file and the line you read it on. Terminals make these clickable, so the reader can audit any node in one keystroke instead of trusting the drawing.

Full worked example, format included: [`EXAMPLE.md`](EXAMPLE.md).

## 4. Read the trace

The tree is evidence. Close by saying what it exposes — the reason a principal engineer asked for it rather than a module diagram.

Hunt these ten, all of them visible in the tree you just drew:

- **Sequential independent I/O** — sibling calls that each cross a boundary and share no input. Round trips paid in series that could be paid at once.
- **Fan-out** — a `× N` node that crosses a boundary. One rendered call, N round trips, and the count is set by user data.
- **Unawaited work on a response path** — a `⇢` node under a request handler. Its failures never reach the caller; name what recovers them, or that nothing does.
- **Boundary inside a lock or transaction** — external calls sitting between an acquire and its release. Hold time is now someone else's latency.
- **Retry without an idempotency key** — a retry branch whose subtree mutates an external system, with no key node on it.
- **Unbounded external call** — a vendor boundary or `↗` seam with no timeout node above it on its path. On a seam it is worse: the far side's slow day becomes your outage.
- **Competing writers** — two `!` nodes writing the same resource from different sections. Name which one the system treats as the source of truth, or that the trace cannot tell.
- **Unrecovered partial write** — a `!` that lands before a later step on the same path can fail, with no compensating call in the failure branch. The system can stop in a state no branch describes.
- **Duplicate work** — the same call reached twice through different branches of one request.
- **Chokepoint** — one leaf that every branch of a section passes through.

Format each finding as: name, the section it lives in, the nodes involved, and the consequence in one line. Rank by consequence, cap at six.

Every finding names nodes that appear in a tree above it and resolves to a line in the source index. A concern you cannot point at nodes for belongs in conversation, not in the block. Two grounded findings beat six padded ones, and none is a valid result — say the trace surfaced none and stop.

## 5. Print

Print the result to the terminal inside a fenced `text` block. Write a file only when asked for one.
