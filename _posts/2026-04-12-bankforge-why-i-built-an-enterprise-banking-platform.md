---
layout: post
title: "BankForge: A Hands-On Deep Dive into Distributed Systems Patterns"
date: 2026-04-12
categories: [engineering]
tags: [java, spring-boot, microservices, kafka, kubernetes, istio, bankforge]
excerpt: "Microservices look clean on architecture diagrams. The hard problems only show up when you actually build one. So I built a banking platform to find out where they hide."
---

Microservices look clean on architecture diagrams. Boxes, arrows, clean boundaries. You've seen the diagram a hundred times.

What the diagram doesn't show: how you move money atomically across a distributed system without 2PC. How you guarantee an event is delivered exactly once when your message broker crashes mid-flight. How you debug a slow request that touched six services before it returned an error. How you secure pod-to-pod traffic without writing auth code in every service.

These are the hard problems. And the only way I know to really understand a hard problem is to build it yourself, from scratch, until it breaks.

So I built **BankForge** — an Australian core banking platform designed as a learning sandbox for enterprise microservices patterns. Four Spring Boot services, a full event streaming pipeline, distributed tracing, a service mesh, and an AI layer that can diagnose production issues autonomously.

## Why Banking

Banking is a good domain for learning distributed systems because the constraints are unforgiving.

Money is not eventually consistent. If I debit your account and the credit to the destination account fails, that's not a "temporary inconsistency to be resolved later" — that's a bug with real consequences. The system has to get atomicity right, even when it spans multiple services and multiple databases.

Banking also has compliance requirements (AUSTRAC threshold reporting in the Australian context), idempotency requirements (duplicate payment requests must not double-charge), and audit trail requirements (every ledger entry must be traceable to its originating event). All of these constraints force you to make the right architectural choices rather than the convenient ones.

The domain is also concrete enough that you always know what "correct" looks like. A transfer either moved the money or it didn't. The ledger either balances or it doesn't.

## The Hard Problems and How I Addressed Each One

### Problem 1: Money Atomicity Across Services

The naive approach: `payment-service` calls `account-service` to debit, then calls `ledger-service` to record the entry, then calls `notification-service` to alert the customer. Each call is synchronous. If ledger-service is down, the transfer fails. If ledger-service times out after the debit but before confirming the record, you've debited the account but have no ledger entry.

**The solution: Local ACID + Choreography Saga**

The money movement itself stays inside a single local transaction in `account-service`:

```
BEGIN TX
  SELECT accounts WHERE id = $source FOR UPDATE  -- pessimistic lock
  SELECT accounts WHERE id = $dest FOR UPDATE
  CHECK: balance >= amount AND account ACTIVE
  UPDATE accounts SET balance = balance - amount WHERE id = $source
  UPDATE accounts SET balance = balance + amount WHERE id = $dest
  INSERT INTO outbox (event_type, payload) VALUES ('TransferInitiated', {...})
COMMIT
```

One database, one transaction, one commit. If anything fails, the entire thing rolls back — no partial debit without credit. The outbox row is written in the same transaction, so if the commit succeeds, the downstream event is guaranteed to follow.

Ledger recording and customer notification happen asynchronously via Kafka. If ledger-service is down, the money has already moved safely — ledger will catch up when it recovers. These are fire-and-forget steps. They don't affect the atomicity of the transfer itself.

### Problem 2: Reliable Event Delivery Without Dual-Write

Here's a subtle failure mode most architectures get wrong.

If you call `kafkaTemplate.send()` inside a database transaction:
- DB commits, Kafka publish fails → event lost, downstream never notified
- Kafka publish succeeds, DB rolls back → phantom event, downstream acts on a transfer that never happened

Both modes silently corrupt your system state.

**The solution: Outbox + Debezium CDC**

Instead of publishing to Kafka directly, services write events to an `outbox` table in the same local transaction as the business operation. Debezium — a CDC (Change Data Capture) tool — reads the PostgreSQL Write-Ahead Log and publishes those rows to Kafka.

```
PostgreSQL ACID TX (account-service):
  money movement + outbox row = committed together

PostgreSQL WAL → Debezium → Kafka
  (async, after commit, exactly this order)
```

The WAL only contains committed data. So Debezium can only publish events that correspond to committed transactions. If Debezium crashes and restarts, it replays from its last committed Kafka offset — no events are lost, no phantom events are created.

The trade-off: your consumers must be idempotent (Debezium guarantees at-least-once, not exactly-once). Ledger-service uses the transfer ID as a deduplication key — `INSERT ... ON CONFLICT DO NOTHING`.

### Problem 3: The Observability Gap

When a distributed system misbehaves, you need to answer: which service is slow? Which hop in the call chain added latency? What was the system doing at the time of the error?

Logs alone don't answer this. Logs are per-service. A slow transfer that touched four services leaves four separate log streams with no shared context linking them.

**The solution: Full observability stack**

```
Services → OpenTelemetry SDK → OTel Collector → Jaeger (traces)
Services → Micrometer → Prometheus scrape (metrics)
Services → ECS-format JSON logs → Promtail → Loki
Grafana → unified dashboards across all three sources
```

Every transfer gets a trace ID that propagates through every service it touches. In Jaeger, you see the full call graph: Kong → payment-service → account-service, with timing for each hop and full span metadata. A LogQL query for that trace ID returns matching log lines from every service, filtered by the same request.

The observability stack tells you *what happened*. The next layer tells you *why*.

### Problem 4: Understanding System Topology

Prometheus can tell you the p99 latency of account-service. It cannot tell you that account-service's latency is caused by its dependency on a shared PostgreSQL instance that payment-service is also hammering, and that the bottleneck propagates upstream through three other services.

Time-series metrics answer "how is X performing right now?" They cannot answer "what is connected to what, and how does a problem in one node propagate through the graph?"

**The solution: Neo4j service graph**

A scheduled ETL job queries Prometheus every 30 seconds for Istio's `istio_request_duration_milliseconds` and `istio_requests_total` metrics — per service pair — and upserts them as edges in Neo4j:

```cypher
MERGE (src:Service {name: $source})
MERGE (dst:Service {name: $target})
MERGE (src)-[r:OBSERVED_CALL]->(dst)
SET r.avg_latency_ms = $avg_latency,
    r.p99_latency_ms = $p99,
    r.error_rate = $error_rate,
    r.updated_at = datetime()
```

Now you can ask questions that PromQL can't express:

```cypher
-- Which service pairs have p99 > 500ms?
MATCH (a)-[r:OBSERVED_CALL]->(b)
WHERE r.p99_latency_ms > 500
RETURN a.name, b.name, r.p99_latency_ms ORDER BY r.p99_latency_ms DESC

-- What's the call path from payment-service to the slowest leaf?
MATCH path = (entry:Service {name: "payment-service"})-[:OBSERVED_CALL*1..3]->(leaf)
RETURN path
```

Prometheus answers "what is metric X right now." Neo4j answers "which services share a common upstream bottleneck."

### Problem 5: Security Without Auth Code in Every Service

The naive approach: every service validates JWTs. Every service has auth middleware. Auth logic is duplicated, must be updated consistently, and becomes a source of bugs.

**The solution: Two-layer gateway (Kong + Istio)**

```
[External] → Kong → [Cluster boundary] → Istio → [Service]
```

Kong handles north-south traffic: validates JWT signatures against Keycloak's JWKS endpoint (cached locally, no per-request Keycloak call), enforces rate limits, strips the `Authorization` header, and injects `X-User-Id` from the JWT `sub` claim. Services see only this injected header — they never handle a raw JWT.

Istio handles east-west traffic: enforces mTLS `STRICT` mode between pods. Every pod-to-pod request requires a valid SPIFFE identity certificate. A pod without a sidecar certificate is rejected. Services write zero TLS code.

The result: all auth and encryption are infrastructure concerns. Service code handles business logic only.

## The Architecture

```
[External Client]
      │ HTTPS + JWT
      ▼
   Kong API Gateway
   (JWT validation, rate limiting, X-User-Id injection)
      │
      ▼
   Istio Service Mesh (mTLS STRICT)
      │
  ┌───┴────────────────────────────┐
  │                                │
  ▼                                ▼
payment-service              account-service
(state machine,              (ACID transfers,
 Redis idempotency)           outbox table)
      │                           │
      └──────────┬────────────────┘
                 │ outbox tables
                 ▼
         Debezium CDC
         (reads PostgreSQL WAL)
                 │
                 ▼
              Kafka
                 │
        ┌────────┴──────────┐
        ▼                   ▼
  ledger-service    notification-service
  (double-entry)    (alerts)

Observability layer (all services):
  OTel → Jaeger (traces)
  Micrometer → Prometheus (metrics)
  Logback → Loki (logs)
  Grafana (dashboards)

Intelligence layer:
  Prometheus ETL → Neo4j (service graph)
  Python MCP server → Claude Desktop (AI RCA)
```

## The Roadmap

The project is structured in five phases, each adding one layer:

| Phase | What it delivers |
|-------|-----------------|
| **1: Core Banking** | Four Spring Boot services, ACID transfers, state machine, Redis idempotency |
| **1.1: CDC Pipeline** | Kafka KRaft + Debezium CDC consuming the outbox, DLT topics, AUSTRAC audit logging |
| **2: Observability** | OTel traces, Prometheus metrics, Loki logs, Grafana dashboards |
| **3: Service Mesh** | Kubernetes (kind) + Istio mTLS + Kong JWT + Keycloak |
| **4: Graph** | Neo4j service graph populated from Prometheus/Istio metrics via ETL |
| **5: AI** | Python MCP server — Claude can query live banking state and run autonomous RCA |

Compose-first (Phases 1–2), then cut over to Kubernetes (Phase 3), then layer the intelligence (Phases 4–5). Each phase has a clear goal, explicit success criteria, and a validation step before proceeding.

## What I'm Actually Validating

I've worked with parts of this stack in production — Kafka at scale, Kubernetes in enterprise environments, distributed tracing. But "worked with" and "built from scratch" are different kinds of understanding.

Building BankForge is how I answer: do I know *why* Outbox + CDC is the right pattern, or do I just know that it is? Can I explain the exact failure modes of dual-write? Can I wire Debezium to PostgreSQL WAL from first principles? Can I implement mTLS in a service mesh without reading a tutorial?

The system becomes evidence that the knowledge is real, not just recalled.

---

*The full series goes deeper into each layer. Next up: the Outbox + CDC pipeline — exactly how reliable event delivery works, and the failure modes it eliminates.*

*[BankForge on GitHub](https://github.com/Harry-Zhao-AU/BankForge)*
