---
layout: post
title: "Choreography vs Orchestration: Why BankForge Uses Choreography and What It Costs You"
date: 2026-04-17
categories: [engineering]
tags:
  [
    java,
    spring-boot,
    kafka,
    microservices,
    distributed-systems,
    saga,
    observability,
    bankforge,
  ]
excerpt: "BankForge uses choreography for its Saga, not orchestration. Real banks do the opposite. Here's why I made that choice, the three failure modes choreography exposes (idempotency, empty rollback, suspension), and how trace ID plus transaction ID stitch the whole distributed flow back together."
---

When you build a distributed payment system, one of the first architectural decisions is how services coordinate during a multi-step business operation. The two approaches — **choreography** and **orchestration** — look similar on a diagram but have radically different failure modes, debugging stories, and operational costs.

BankForge uses choreography. Real banks almost universally use orchestration. This post explains the trade-off, what choreography forces you to solve explicitly, and how distributed tracing lets you reconstruct the full picture when something goes wrong.

---

## The Two Coordination Models

### Choreography

No service is in charge. Each service reacts to events and emits new ones. The flow emerges from the combination of listeners and publishers.

```
Client → payment-service → REST → account-service
                                         │ (debit+credit+outbox in 1 TX)
                                         ▼
                               Debezium CDC → Kafka
                                         │
                              ┌──────────┴───────────┐
                              ▼                       ▼
                       ledger-service         notification-service
                    (posts entries + outbox)
                              │
                     Debezium CDC → Kafka
                              │
                              ▼
                       payment-service ← banking.transfer.confirmed
                       (state → CONFIRMED)
```

payment-service never tells ledger-service "go post these entries." ledger-service subscribes to the event topic and acts on its own. Nobody has a global view of the in-progress saga.

### Orchestration

One service drives the sequence. It issues explicit commands to each participant, waits for replies, and handles failures centrally.

```
Client → payment-orchestrator
              │
              ├─ command → account-service: "execute debit+credit"
              │            ← reply: "done" or "insufficient funds"
              │
              ├─ command → ledger-service: "post entries"
              │            ← reply: "posted"
              │
              └─ state → CONFIRMED
```

The orchestrator has a complete picture at every step. It knows which step failed. It decides whether to compensate.

---

## Why Real Banks Use Orchestration

Real banks operate under regulatory constraints that choreography handles poorly.

**Auditability.** APRA, RBA, and AUSTRAC can ask: _what was the exact sequence for transfer T-12345, who authorised step 3, and why did it fail?_ With choreography, the answer is scattered across four service logs and four event topics. With an orchestrator, the saga state machine has the full history in one place — every transition, every command, every reply, every compensation.

**Determinism.** Choreography relies on "services will react correctly to events in order." Orchestration guarantees it: each step is explicitly triggered and explicitly confirmed before the next step starts. For operations that move money, the difference between "should work in practice" and "guaranteed by design" matters.

**Compensation precision.** When a choreography step fails after the event has already been published, reversing is complex — you publish a compensating event and hope every downstream subscriber correctly handles it. An orchestrator calls each compensating step directly and waits for confirmation.

**ISO 20022 / NPP (Australia's New Payments Platform).** Real payment flows are explicit state machines. The NPP processes every transfer as a sequenced workflow: validation → clearing → settlement → confirmation. Each step is a named, logged transaction. There is no ambiguity about what happened.

---

## Why BankForge Uses Choreography

BankForge is a learning sandbox, not a production payment processor. Choreography is the more interesting thing to build because it exposes failure modes that orchestration hides.

With orchestration, if you forget to handle a failure, the orchestrator's central error handler catches it. With choreography, every gap in your resilience design becomes a stuck saga. You are forced to explicitly solve:

- **Idempotency** — at-least-once delivery means duplicates are guaranteed, not exceptional
- **Empty rollback** — compensating a step that may not have executed
- **Suspension** — a step that never fails, never succeeds, just stops

These are the same problems orchestration handles internally. Choreography makes you see them directly. That is the point.

(In practice, Kafka/CDC is the right pattern for the edges of a real banking system — async fan-out to analytics, reporting, fraud ML pipelines, customer notifications. Not for the core debit/credit/post sequence.)

---

## The Three Saga Failure Modes

### 1. Idempotency

In a choreography saga with at-least-once Kafka delivery, every consumer must tolerate receiving the same event more than once. If ledger-service crashes after writing ledger entries but before its consumer offset is committed, the event is redelivered on restart. Without idempotency, a customer gets charged twice.

BankForge handles this at two layers.

**Layer 1: HTTP entry point (Redis + DB constraint)**

When a client submits a transfer, payment-service checks Redis first:

```java
Optional<String> cached = idempotencyService.getCached(request.idempotencyKey());
if (cached.isPresent()) {
    return deserialize(cached.get()); // return the original response — no re-execution
}
```

Redis uses `SET NX EX` — atomic first-write semantics. The first request sets the key, concurrent duplicates miss the set and return the cached response.

Redis is the fast path. The fallback is a database unique constraint on `idempotency_key`. If two requests race past the Redis check before either has written to Redis, exactly one will commit the DB row; the other gets a `DataIntegrityViolationException`, which is caught and resolved by returning the existing record:

```java
try {
    transfer = transferStateService.createAndStart(request);
} catch (DataIntegrityViolationException e) {
    transfer = transferRepository.findByIdempotencyKey(request.idempotencyKey())
            .orElseThrow(...);
    return buildAndCacheResponse(transfer, request.idempotencyKey());
}
```

**Layer 2: Kafka consumer (DB existence check)**

ledger-service consumes `banking.transfer.events`. On redelivery, the transfer ID already has a ledger entry. The guard prevents duplicate posting:

```java
if (ledgerEntryRepository.existsByTransferId(transferId)) {
    return; // already processed — outbox row written, Debezium already published confirmation
}
```

This is backed by a UNIQUE constraint on `transfer_id` in the ledger table. Even if the existence check races (two concurrent consumers), the DB constraint ensures only one write wins.

---

### 2. Empty Rollback

The empty rollback is a subtle saga failure mode: your compensation logic runs, but there is nothing to compensate.

In BankForge, the scenario looks like this:

```
payment-service calls account-service.executeTransfer()
                                    │
                          Network timeout ← ???
                          Did money move or not?
```

A network timeout means the HTTP call did not complete. But "did not complete" has two possible causes:

- The request never reached account-service (money did not move)
- account-service completed the debit/credit but the response was lost in transit (money moved)

If you blindly trigger a reversal in the catch block, you may be reversing a transfer that never happened — creating a negative balance from nowhere.

BankForge solves this with a flag:

```java
boolean transferExecuted = false;
try {
    accountServiceClient.executeTransfer(...);
    transferExecuted = true;           // only set AFTER a successful return

    transfer = transferStateService.advanceToPosting(transfer.getId());

} catch (Exception e) {
    if (transferExecuted) {
        // We got a successful response — money definitely moved
        // Safe to reverse
        accountServiceClient.reverseTransfer(...);
    }
    // If transferExecuted is false: executeTransfer() itself threw
    // Outcome is uncertain — do NOT attempt reversal
    // Flag for manual review, cancel the saga record

    transfer = transferStateService.cancel(transfer.getId(), e.getMessage());
}
```

`transferExecuted` is only set to `true` after `executeTransfer()` returns successfully. If `executeTransfer()` itself throws — whether due to network timeout, connection refused, or account-service being down — the flag stays `false` and no reversal is attempted.

This is the correct behaviour. A reversal on an un-executed transfer would debit an account that was never credited. The transfer is cancelled and flagged; a human reviews the uncertain cases.

---

### 3. Suspension

Suspension is the choreography-specific failure mode that orchestration handles implicitly. A saga step neither succeeds nor fails — it just stops producing progress.

In BankForge, a transfer reaches `POSTING` state when account-service has confirmed the money moved. ledger-service is supposed to consume the event, write ledger entries, and publish `banking.transfer.confirmed` back to payment-service. If ledger-service crashes after writing entries but before its consumer offset advances, the event is redelivered (idempotency handles the duplicate). But if the Debezium connector stalls, or the Kafka broker is partitioned, `banking.transfer.confirmed` is never published. The transfer sits in `POSTING` indefinitely.

Nobody fails. Nobody calls an error handler. The saga is just stuck.

BankForge's answer is `HungTransferDetector` — a scheduled watchdog running every 60 seconds:

```
PAYMENT_PROCESSING > 5 min   → cancel via FSM compensation path
                                (money likely never moved — safe to cancel)

POSTING > 15 min             → mark FAILED directly (bypass FSM)
                                (ledger entries are written — do NOT reverse
                                 or you double-reverse a balanced ledger)
```

The two timeouts reflect the different risk. PAYMENT_PROCESSING is pre-money-movement — cancellation is safe. POSTING is post-money-movement — the books are already balanced; the only missing piece is the state machine confirmation. Reversing would corrupt the ledger.

```java
// POSTING hung: bypass FSM — intentional
transfer.setState(TransferState.FAILED);
transfer.setErrorMessage("LEDGER_CONFIRM_TIMEOUT -- manual review required");
transferRepository.save(transfer);
// No compensation, no reversal event
```

This is the operational cost of choreography made explicit: you need a watchdog because there is no coordinator tracking global progress.

---

## Tracing the Distributed Flow

The hardest part of debugging a choreography saga is correlating events across services. A transfer touches four services across REST calls and Kafka messages. By the time something goes wrong, the relevant spans and log lines are spread across four separate telemetry streams.

BankForge uses two identifiers in combination:

| Identifier                 | What it is           | How it propagates                                                     |
| -------------------------- | -------------------- | --------------------------------------------------------------------- |
| **Trace ID**               | OTel W3C traceparent | Auto-propagated by Spring OTel through HTTP headers and Kafka headers |
| **banking.transaction.id** | The transfer UUID    | Manually propagated via W3C Baggage; set as an OTel span attribute    |

### Trace ID: The Technical Chain

The trace ID links every span in the distributed call — REST hops, Kafka publishes, Kafka consumes. It's the OTel standard `traceparent` header, propagated automatically by Spring's OTel instrumentation.

In Jaeger, a single trace ID shows you:

```
payment-service: POST /transfers [50ms]
  └── account-service: POST /internal/transfers [30ms]
        └── (Debezium CDC → Kafka: async, separate trace unless linked)
ledger-service: @KafkaListener banking.transfer.events [12ms]
  └── (Debezium CDC → Kafka: writes outbox row)
payment-service: @KafkaListener banking.transfer.confirmed [3ms]
```

The Kafka segments are linked across async boundaries via the `traceparent` column in the outbox table. When ledger-service writes its outbox row, it captures the current span context:

```java
String outTraceparent = spanCtx.isValid()
    ? "00-" + spanCtx.getTraceId() + "-" + spanCtx.getSpanId() + "-01"
    : null;

LedgerOutboxEvent outboxRow = LedgerOutboxEvent.builder()
    ...
    .traceparent(outTraceparent)
    .build();
```

Debezium's `EventRouter` SMT lifts the `traceparent` column to a Kafka message header. payment-service's `TransferConfirmationListener` receives it, restores the OTel context, and the span is stitched back into the original trace — one continuous trace across the entire async Kafka boundary.

### banking.transaction.id: The Business Chain

The trace ID solves the technical correlation — it links spans. But it doesn't tell you which business transfer a span belongs to. `banking.transaction.id` is the transfer UUID set as an OTel span attribute on every span in the flow.

The challenge is that account-service receives the transfer request independently via REST — it doesn't inherently know which `Transfer` entity in payment-service this corresponds to. Without explicit propagation, account-service would generate a random UUID for its outbox event, and ledger-service's confirmation would contain a UUID that payment-service can't match.

BankForge propagates the transfer ID via W3C Baggage:

```java
// payment-service: after creating the Transfer record
try (Scope ignored = Baggage.current().toBuilder()
        .put("banking.transaction.id", transfer.getId().toString())
        .build()
        .makeCurrent()) {
    accountServiceClient.executeTransfer(...);
}
```

```java
// account-service: reads the baggage to reuse the same UUID
String baggageTxnId = Baggage.current().getEntryValue("banking.transaction.id");
UUID transferId = (baggageTxnId != null)
    ? UUID.fromString(baggageTxnId)
    : UUID.randomUUID();
Span.current().setAttribute("banking.transaction.id", transferId.toString());
```

```java
// ledger-service: reads from Kafka header via restored OTel context
Span.current().setAttribute("banking.transaction.id", transferId.toString());
```

The result: every span across all four services has the same `banking.transaction.id` attribute. In Jaeger, filtering on this attribute shows the complete distributed trace for a single transfer. In Loki (structured logs), the same query returns every log line across every service for that transfer.

### What You Can Answer

With trace ID + banking.transaction.id, you can answer:

```
"Transfer T-9b4f2... was stuck in POSTING for 14 minutes"

→ Jaeger: filter spans by banking.transaction.id = "9b4f2..."
  payment-service span: entered POSTING at 14:32:01
  ledger-service span: consumed banking.transfer.events at 14:32:03, wrote ledger entries
  ledger-service span: wrote outbox row at 14:32:03
  (gap — no banking.transfer.confirmed span)

→ Loki: same banking.transaction.id filter
  ledger-service: "Outbox row written — Debezium will publish confirmation" at 14:32:03
  (no Debezium publication log after that)

→ Diagnosis: Debezium connector stalled on the ledger_outbox_event table
  HungTransferDetector fired at 14:47:02, marked FAILED
```

Without the business-level ID, you can observe the Debezium stall — but you can't connect it to a specific transfer or determine which customers are affected.

---

## The Complete Picture

| Layer               | Problem solved                                     | BankForge implementation                                                                         |
| ------------------- | -------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| Saga (choreography) | Multi-step coordination across services            | Event-driven: CDC → Kafka → listeners                                                            |
| Idempotency         | At-least-once delivery produces duplicates         | Redis (fast path) + DB constraint (fallback) + `existsByTransferId` in consumers                 |
| Empty rollback      | Compensating a step that may not have executed     | `transferExecuted` flag — reversal only if HTTP call confirmed successful                        |
| Suspension          | Saga step stalls without failing                   | `HungTransferDetector` — per-state timeouts with different compensation actions                  |
| Tracing             | Correlating events across async service boundaries | OTel `traceparent` in Kafka headers via outbox column + `banking.transaction.id` via W3C Baggage |

Choreography forces every one of these into the open. The empty rollback problem exists in orchestration too — but the orchestrator's error handler catches it in one place. In choreography, every service that participates in a compensatable step must handle it explicitly.

That's the real cost of choreography: not the complexity of the design, but the discipline required to close every failure mode that the absence of a coordinator leaves open.

---

_BankForge is an open-source Australian core banking platform built as a learning sandbox for enterprise microservices patterns. The source is on GitHub._
