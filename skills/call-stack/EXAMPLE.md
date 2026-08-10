# Worked example

One finished run of [`call-stack`](SKILL.md) against an invented orders service — a checkout endpoint, a cron sweep, and a background processor that charges a card and ships the order. Every symbol here is fictional; the point is the shape.

It shows the levers in use: entry points as sections (1, 2), a shared subtree hoisted into its own section that both callers point at (3), a deep pipeline split out (4), conditionals rendered as question nodes with their outcomes beneath (5), and a re-entry path where most of the work is skipped (6).

Read the markers first — they carry what the tree cannot say on its own. `checkStock() × per line item` is a fan-out hiding behind one node. `⇢ OrderProcessor.run()` is dispatched and never awaited, so nothing it throws reaches the checkout response. `! writes orders` separates the calls that change durable state from the ones that only read it. `↗ notifications-svc` is the honest edge of this repo — the tree continues in code that lives somewhere else, so the branch is cut and says so.

Then the `SOURCE INDEX` at the foot: every claim above it is one click from the line it came from. And `FINDINGS` last, each one pointing back at nodes already drawn.

```text
ORDERS SERVICE — RUNTIME CALL STACK
===================================

1. CHECKOUT SUBMISSION
----------------------

CheckoutForm.onSubmit()
└─ POST /api/checkout
   └─ handleCheckout()
      ├─ request.json()
      ├─ checkoutSchema.safeParse()
      ├─ Session.current()
      │  └─ Redis
      ├─ Cart.load(sessionId)
      │  └─ Postgres
      ├─ Inventory.reserve(lineItems)
      │  ├─ checkStock()  × per line item
      │  │  └─ Postgres
      │  └─ writeHold(15-minute TTL)  ! writes holds
      │     └─ Redis
      ├─ Order.create()
      │  ├─ normalizeAddress()
      │  ├─ randomUUID()
      │  └─ OrderStore.insert(status = Pending)  ! writes orders
      │     └─ Postgres
      ├─ enqueue()
      │  └─ ⇢ OrderProcessor.run(orderId)
      └─ HTTP 201 / 409 / 422


2. SCHEDULED SWEEP
------------------

Cron — every 5 minutes
└─ GET /api/sweep
   └─ handleSweep()
      ├─ verify CRON_SECRET
      ├─ Inventory.expireHolds()
      │  ├─ Redis scan
      │  └─ release stock older than 15 minutes  ! writes holds
      │     └─ Postgres
      ├─ OrderStore.stuck(older than 1 hour)
      │  └─ Postgres
      ├─ select oldest Pending Order
      └─ enqueue()
         └─ ⇢ OrderProcessor.run(orderId)


3. SHARED PROCESSOR
-------------------

OrderProcessor.run(orderId)
├─ OrderStore.get(orderId)
├─ payment already captured?
│  └─ YES → deliverReceipt()
│     └─ Mailer.send()  ↗ notifications-svc
│        └─ … deeper not traced
├─ acquireLock(orderId)
│  └─ Redis lease
├─ OrderStore.beginAttempt()  ! writes orders
├─ create AbortController
├─ start 60-second timeout
├─ fulfill(order, signal)
│  └─ runPipeline()
├─ OrderStore.awaitReceipt(result)
│  └─ persist Awaiting Receipt  ! writes orders
├─ deliverReceipt()
│  ├─ Mailer.send()  ↗ notifications-svc
│  │  └─ … deeper not traced
│  └─ persist Completed  ! writes orders
└─ releaseLock()
   └─ Redis


4. FULFILLMENT PIPELINE
-----------------------

fulfill()
└─ runPipeline()
   ├─ Inventory.commit(holdId)  ! writes holds, stock
   │  └─ Postgres
   │
   ├─ Payment.charge()
   │  ├─ buildIdempotencyKey(orderId)
   │  ├─ Stripe PaymentIntent  !
   │  └─ declined?
   │     └─ throw PaymentDeclinedError
   │        └─ Processor converts to Payment Failed
   │
   ├─ Promise.all()
   │  ├─ Tax.calculate()
   │  │  └─ Avalara
   │  └─ Shipping.quote()
   │     ├─ carrierRates()  × per carrier
   │     │  └─ EasyPost
   │     └─ pickCheapest()
   │
   ├─ Label.purchase()  !
   │  └─ EasyPost
   ├─ Invoice.render()
   │  └─ PDF buffer
   ├─ Invoice.store()  ! writes invoices
   │  └─ S3
   │
   └─ assemble()
      └─ Fulfilled Order
         ├─ tracking number
         └─ invoice URL


5. OUTCOME BRANCHES
-------------------

fulfill()
├─ charge captured and label bought
│  └─ Fulfilled Order
├─ PaymentDeclinedError
│  ├─ Inventory.release()
│  └─ Payment Failed
├─ carrier unreachable
│  └─ Awaiting Shipping — retried by sweep
├─ deterministic failure
│  └─ Order Failed
└─ transient failure
   ├─ attempts < 3 → return to Pending
   └─ attempts exhausted → Order Failed


6. RECEIPT RETRY
----------------

OrderProcessor.run(orderId)
└─ payment already captured
   ├─ skip Inventory.commit()
   ├─ skip fulfill()
   ├─ deliverReceipt()
   │  └─ ↗ notifications-svc
   ├─ failure → remain Awaiting Receipt
   └─ success → Completed


SOURCE INDEX
------------

1. CHECKOUT SUBMISSION    src/api/checkout.ts:14
   Cart.load               src/cart/store.ts:88
   Inventory.reserve       src/inventory/reserve.ts:31
   Order.create            src/orders/create.ts:22
   enqueue                 src/queue/enqueue.ts:9

2. SCHEDULED SWEEP        src/api/sweep.ts:11
   Inventory.expireHolds   src/inventory/reserve.ts:104
   OrderStore.stuck        src/orders/store.ts:141

3. SHARED PROCESSOR       src/queue/processor.ts:27
   acquireLock             src/lib/lock.ts:16
   deliverReceipt          src/orders/receipt.ts:8

4. FULFILLMENT PIPELINE   src/fulfillment/pipeline.ts:19
   Payment.charge          src/payments/stripe.ts:44
   Shipping.quote          src/shipping/quote.ts:30
   Invoice.render          src/invoices/render.ts:12
```

Findings:

```text
FINDINGS
--------

1. Lock held across three external calls — section 3, 4
   acquireLock() spans Stripe, EasyPost, and S3 before
   releaseLock(). Redis lease and pipeline timeout are both
   60s, so a slow carrier can expire the lease mid-write and
   let the sweep start a second processor on the same order.

2. Sequential independent I/O — section 4
   Payment.charge → Label.purchase → Invoice.store run in
   series. Label and Invoice share no input; only Tax and
   Shipping are parallelized today.

3. Unawaited dispatch on the request path — section 1
   ⇢ OrderProcessor.run() means nothing it throws reaches the
   201. The 5-minute sweep is the only recovery, so a failure
   is invisible to the customer for up to five minutes.

4. Fan-out on checkout — section 1
   checkStock() × per line item, one Postgres round trip each.
   A 40-item cart is 40 queries inside the request.

5. Retry without an idempotency key — section 5
   Transient failure returns the order to Pending, and the
   sweep re-enters at Payment.charge. buildIdempotencyKey()
   covers the Stripe call; Label.purchase() has no equivalent,
   so a retry after a carrier timeout can buy a second label.

6. Unrecovered partial write — section 4, 5
   Inventory.commit() and the Stripe charge both land before
   Label.purchase() can fail. The decline branch compensates
   with Inventory.release(); the carrier-unreachable branch
   does not. That path ends with money captured, stock
   committed, and nothing shipped.
```

Primary dependency chain:

```text
HTTP routes
  → Order + Inventory
  → OrderProcessor
  → Fulfillment pipeline
  → Payment / Shipping / Tax adapters
  → Stripe + EasyPost + Avalara
  → Postgres + Redis + S3
  → Mailer
  ↗ notifications-svc (separate repo)
```
