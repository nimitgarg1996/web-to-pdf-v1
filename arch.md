# AI Document Processing Platform — Engineering Plan

**Version:** 1.0  
**Status:** Approved for Component Design  
**Scope:** L7 AI Reverse Proxy + Document Processing Pipeline Architecture

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Root Cause Analysis](#2-root-cause-analysis)
3. [Architecture Overview](#3-architecture-overview)
4. [Architecture Pattern Decision](#4-architecture-pattern-decision)
5. [Full System Flow](#5-full-system-flow)
6. [Phase 1 — Infrastructure](#6-phase-1--infrastructure)
7. [Phase 2 — Proxy Core](#7-phase-2--proxy-core)
8. [Phase 3 — Distributed Circuit Breaker](#8-phase-3--distributed-circuit-breaker)
9. [Phase 4 — Token Budget Admission Control](#9-phase-4--token-budget-admission-control)
10. [Phase 5 — Priority Surge Queue](#10-phase-5--priority-surge-queue)
11. [Phase 6 — Observability](#11-phase-6--observability)
12. [Failure & Retry Strategy](#12-failure--retry-strategy)
13. [Header Contract](#13-header-contract)
14. [Bedrock AIP & Service Tiers](#14-bedrock-aip--service-tiers)
15. [What Was Deliberately Not Built](#15-what-was-deliberately-not-built)
16. [Further Optimisations](#16-further-optimisations)

---

## 1. Problem Statement

A document processing platform allows users to upload PDFs and scanned documents. Java services enqueue documents into SQS. A Lambda consumer reads each message and invokes a Step Function. The Step Function orchestrates OCR, Summarization, QnA, and Doc Extraction — all of which call AWS Bedrock via an Application Inference Profile (AIP).

**Observed failure modes:**

- Bedrock returns `ThrottlingException` under load
- Lambda functions sit in exponential backoff retry loops, burning execution time and concurrency
- 50 documents uploaded simultaneously fan out into thousands of concurrent Bedrock calls
- No feedback mechanism between the Bedrock boundary and the upstream pipeline
- Retry storms: throttled requests retry in near-synchrony, compounding the failure

---

## 2. Root Cause Analysis

The throttling is not primarily a race condition problem. It is an **uncontrolled fan-out explosion** with no concurrency governor anywhere in the chain.

### The blast radius of a single upload event

```
50 docs uploaded
  → 50 SQS messages          (no throttle)
    → 50 concurrent Lambdas   (no throttle)
      → 50 Step Functions      (all running simultaneously)
        → OCR: ~4 concurrent Bedrock calls per doc = 200 calls
        → Summarization: 50 calls
        → QnA + Doc Extraction (parallel): 100 calls
          = 350+ concurrent Bedrock calls hitting AIP limit
            → ThrottlingException
              → Lambda retries in-place
                → Burns execution time
                  → Retry storm
```

### Why naive fixes don't work

| Fix Attempted | Why It Fails |
|---|---|
| Lambda reserved concurrency = 1 | Lambda exits immediately after invoking Step Function. Concurrency=1 serialises starts, not executions. 50 Step Functions still run concurrently in the background |
| Retry inside Lambda / Step Function | Burns compute waiting. Lambda timeout is 15 minutes max. Eventually exhausts retries, work is lost |
| Fail fast → requeue into SQS immediately | Lambda picks all messages up again immediately. Creates a tight infinite thrashing loop — same throttling, just at SQS layer now |
| RPM-based rate limiting | LLM calls take 30–90 seconds. RPM is meaningless when request duration is this variable |
| Counting requests as admission control | Output tokens are non-deterministic. You cannot predict TPM consumption upfront purely from request counts |

### The real governing unit

Bedrock AIP is governed by **TPM — Tokens Per Minute**, not RPM. For Claude 3.7+ models, the burndown rate is **5× for output tokens**: 1 output token consumes 5 tokens from the TPM quota. This is deducted upfront based on `max_tokens` at request start, then adjusted during processing, and replenished at end for unused tokens.

**Implication:** A Lambda setting a generous `max_tokens` "just in case" burns TPM quota before a single output token is generated, reducing the number of concurrent requests that can be admitted.

---

## 3. Architecture Overview

### Core principle

> Design the system to respect the bottleneck, not fight it.

Bedrock AIP is the narrow pipe. The solution is not to widen it arbitrarily — it is to put a **reservoir before it** that absorbs excess and feeds work through at a controlled rate, while broadcasting the current state of the border to every caller in the system.

### Two complementary solutions

| Layer | Solution | What It Solves |
|---|---|---|
| Pipeline | SQS as durable reservoir + fail-fast + exponential backoff with jitter | Wasted compute, retry storms, work loss |
| Gateway | L7 AI Reverse Proxy with distributed circuit breaker | Shared border awareness, admission control, priority routing, centralized auth |

---

## 4. Architecture Pattern Decision

### Hybrid: Orchestration + Event-Driven Choreography

Pure orchestration (Step Functions throughout) creates tight coupling. Pure choreography has no coordinator to enforce sequencing where it is genuinely required.

**Decision:** Use the right tool for each stage.

| Pattern | Applied Where | Reason |
|---|---|---|
| Orchestration (Step Functions) | OCR → Summarization → OpenSearch write | Strict sequential dependency. Summarization cannot start until OCR completes. QnA cannot fire until summary is indexed in OpenSearch |
| Choreography (EventBridge) | Summary complete → QnA + Doc Extraction | True independence. Neither stage depends on the other. Adding new downstream consumers tomorrow requires zero changes to existing services — just a new EventBridge rule |

### Why pure choreography was rejected for the sequential stages

When Step Functions writes the summary to OpenSearch and emits an EventBridge event, both QnA and Doc Extraction need the summary to already be indexed before they can run. In choreography, there is no coordinator to enforce this guarantee. Step Functions owns this sequencing explicitly and correctly.

---

## 5. Full System Flow

```
User uploads documents (web app)
  ↓
Java service → SQS queue (durable reservoir)
  ↓
Lambda consumer
  - Reads SQS message
  - Invokes Step Function
  - Exits immediately (no retry logic here)
  ↓
Step Function per document
  │
  ├── Stage 1: OCR Lambda
  │     - Processes each page
  │     - Writes each page output to S3 (checkpoint)
  │     - On retry: skips pages already in S3
  │
  ├── Stage 2: Summarization Lambda
  │     - Pulls OCR output from S3
  │     - Chunks and sends to LLM
  │     - Stores summary in OpenSearch
  │
  └── Stage 3: Emit EventBridge event
        Payload: { client-id, doc-id }
              ↓
        EventBridge rule
          ├── → QnA Lambda
          │     RAG pipeline, queries OpenSearch with client-id + doc-id
          │
          └── → Doc Extraction Lambda
                Pydantic model extraction, queries OpenSearch with client-id + doc-id

All LLM calls (from every Lambda) route through:
  → L7 AI Reverse Proxy → Bedrock AIP → Claude / Nova

Chatbot (separate entry point):
  → L7 AI Reverse Proxy (Tier 1) → Bedrock AIP
```

---

## 6. Phase 1 — Infrastructure

### ECS Fargate (Proxy Compute)

- Serverless container cluster in a **private subnet**
- No public IP exposure
- Auto-scaling based on request queue depth
- Multiple tasks running concurrently for availability

**Critical note:** Because multiple Fargate tasks run concurrently, any state that the proxy depends on (circuit breaker flags, surge queue order) must be stored in Redis, not in task-local memory. In-memory state diverges across instances.

### ElastiCache Redis

- Deployed in the same VPC as Fargate
- Single source of truth for:
  - Circuit breaker state (CLOSED / OPEN / HALF-OPEN)
  - Priority surge queue ordering
  - Token budget tracking
  - TTL-based flags
- Use Redis Lua scripts for atomic check-and-set operations where needed

### Bedrock AIP (Application Inference Profile)

- All real-time LLM traffic targets the **AIP ARN**, not raw model IDs
- Enables per-tenant cost attribution via resource tags
- Enables cross-region routing to be layered on later without changing proxy logic
- AIP is separate from Batch Inference — AIP is for real-time on-demand traffic only

### OpenSearch

- Stores document summaries
- Powers RAG pipeline for QnA Lambda
- EventBridge event is only emitted after OpenSearch write confirms success

### EventBridge

- Choreography bus between Step Function completion and downstream consumers
- Event payload: `{ client-id, doc-id }` — stateless, sufficient for consumers to query OpenSearch independently
- Adding new downstream consumers = new EventBridge rule, zero changes to existing services

---

## 7. Phase 2 — Proxy Core

### Runtime Decision

**Recommended: Go or Rust/Tokio**

Do not use Python/FastAPI. FastAPI runs on a single-threaded event loop per worker. Under sustained load, tail latency variance is unacceptable for a system whose primary value proposition is predictable, low-latency arbitration.

| Runtime | Pros | Cons |
|---|---|---|
| Rust/Tokio | Best performance, memory safety, zero-cost abstractions | Steeper learning curve, slower development velocity |
| Go | Excellent concurrency primitives, mature HTTP/2 libraries, fast compile times, strong AWS SDK | Slightly higher memory overhead than Rust |
| Python/FastAPI | Familiar, fast to prototype | Single-threaded event loop, tail latency variance under load |

**Recommendation:** Go if team doesn't have Rust expertise. Rust if performance is the primary constraint.

### Connection Pooling

- Maintain **warm, long-lived HTTP/2 connections** from proxy to Bedrock API
- Eliminates TLS handshake and TCP connection setup latency for every request from ephemeral Lambda callers
- HTTP/2 multiplexing allows multiple in-flight requests on a single connection

### Stateless Routing

- Tenant ID / Service ID → Bedrock AIP ARN mapping
- Seeded from **SSM Parameter Store or AppConfig** at container startup
- TTL-based refresh (e.g., every 5 minutes) to pick up AIP ARN changes
- Define a consistency SLA: changes to AIP ARNs may take up to TTL duration to propagate
- Mapping stored in task-local memory — read-only, safe to diverge briefly across instances

### Authentication Centralization

**This is a major secondary value proposition of the proxy.**

- AWS credentials, Bedrock API keys, and AWS SigV4 request signing live in the proxy
- No Lambda or service manages LLM API auth directly
- Worker Lambdas carry no LLM API concerns — containers are lighter, simpler, and easier to rotate credentials for
- IAM role for proxy Fargate task has Bedrock invoke permissions; worker Lambdas do not

### `max_tokens` Enforcement

The proxy enforces per-route `max_tokens` caps. This is not optional — it directly governs TPM quota consumption.

**Why this matters (Claude 3.7+ burndown mechanics):**

At the start of a request, Bedrock deducts `max_tokens × 5` from your TPM quota upfront. A summarization Lambda setting `max_tokens: 4096` "just in case" burns 20,480 TPM quota units before generating a single output token. This prevents other concurrent requests from being admitted.

**Per-route `max_tokens` caps:**

| Route | Rationale |
|---|---|
| OCR (per page) | Short structured extraction, predictable output length |
| Summarization | Longer output, but bounded by document structure |
| Doc Extraction | Structured Pydantic output, highly predictable size |
| Chatbot QnA | Conversational, moderate ceiling |

Callers may optionally pass `X-Max-Tokens` to request a higher ceiling. Proxy enforces the per-route maximum as an absolute ceiling regardless.

---

## 8. Phase 3 — Distributed Circuit Breaker

### Purpose

The circuit breaker is the **shared nervous system** of the distributed system. It sits at the Bedrock border, observes the error rate, and broadcasts the current border state to every caller — SQS-triggered Lambdas, the chatbot, future services — all of them.

Without this, each caller discovers throttling independently from Bedrock. With this, the system self-regulates: callers check the shared state before making a call and act accordingly.

### State Machine

```
                    error rate > threshold
         CLOSED ─────────────────────────────► OPEN
           ▲                                     │
           │                                     │ TTL expires (e.g. 60s)
           │                                     ▼
           │   canary succeeds             HALF-OPEN
           └───────────────────────────────────── │
                                                  │ canary fails
                                                  │ (reset TTL, back to OPEN)
                                                  ▼
                                               OPEN (TTL reset)
```

| State | Behaviour |
|---|---|
| CLOSED | Bedrock healthy. All requests admitted normally |
| OPEN | Error rate exceeded threshold. All requests fast-failed immediately. Proxy returns 503 |
| HALF-OPEN | TTL expired. Exactly one canary request is allowed through to probe Bedrock. All other requests are still fast-failed |

### Why HALF-OPEN is non-negotiable

Without HALF-OPEN, TTL expiry immediately opens the floodgates. Every request that was queued or requeued during the OPEN period fires simultaneously. This recreates the exact thundering herd that caused the OPEN state in the first place.

HALF-OPEN ensures the system verifies Bedrock has recovered before admitting traffic.

### Redis State Storage

```
Key:    circuit:bedrock:{aip-arn}
Value:  { state: "OPEN" | "CLOSED" | "HALF_OPEN", updated_at: timestamp }
TTL:    Set on OPEN state only (e.g. 60 seconds)
```

Atomic state transitions use Redis Lua scripts to avoid TOCTOU race conditions on the state itself.

### Caller Behaviour on Each State

| State | Caller Action |
|---|---|
| CLOSED | Proceed to token budget check, then dispatch |
| OPEN | Fail immediately. Return 503. Worker Lambda requeues to SQS with backoff |
| HALF-OPEN | Proxy admits canary only. All other callers receive 503 and requeue |

---

## 9. Phase 4 — Token Budget Admission Control

### Purpose

Before dispatching any large request to Bedrock, the proxy estimates its TPM cost and decides whether to admit it now or defer it. This is the layer that makes admission control intelligent rather than purely reactive.

### `CountTokens` Pre-flight

- Bedrock provides a `CountTokens` API that returns the exact token count for a given input
- **It is free — no inference charge**
- Called by the proxy before admitting summarization and doc extraction requests
- Not called for chatbot requests — latency cost of the extra round-trip is unacceptable for Tier 1

### Budget Estimation Formula

```
projected_tpm_cost = input_tokens + (max_tokens × 5)
```

If `projected_tpm_cost` would push current TPM consumption above a configurable threshold stored in Redis, the request is deferred — either queued in the surge queue (Tier 1) or returned as 429 to be requeued in SQS (Tier 2).

### Admission Decision Tree

```
Incoming request
  │
  ├─ Circuit breaker check
  │    OPEN → fail fast → 503 → SQS requeue with backoff
  │
  ├─ Tier 1 (chatbot)?
  │    → skip CountTokens (latency sensitive)
  │    → set service_tier: priority
  │    → dispatch directly
  │
  └─ Tier 2 (document pipeline)?
       → call CountTokens (free, no inference charge)
       → estimate projected TPM cost
       → within budget? → set service_tier: flex → dispatch
       → over budget?   → return 429 → SQS requeue with backoff
```

---

## 10. Phase 5 — Priority Surge Queue

### Purpose

A short-lived, ordered buffer for **transient pressure spikes**. Specifically designed for the chatbot — a live user waiting for a response cannot be requeued into SQS. The surge queue holds their request briefly while capacity recovers.

This is not a general-purpose queue. It is a shock absorber for seconds-long pressure spikes, not sustained throttling (which SQS handles).

### Properties

| Property | Value |
|---|---|
| Storage | Redis-backed. Shared across all Fargate instances |
| Ordering | Priority queue. Tier 1 inserted at front. Tier 2 inserted at back |
| TTL per request | Short (e.g. 5 seconds). Requests that sit too long are evicted, not silently stuck |
| Release mechanism | Micro-jitter on release. When capacity frees up, requests are not released simultaneously |
| Eviction policy | When queue is full, Tier 2 requests are evicted first to make room for Tier 1 |

### `service_tier` Injection

The proxy sets the `service_tier` parameter on every Bedrock API call based on the caller's `X-Priority` header:

| Caller | `X-Priority` | `service_tier` injected |
|---|---|---|
| Chatbot | Tier 1 | `priority` |
| Document pipeline | Tier 2 | `flex` |

Bedrock's own infrastructure handles the actual prioritization. The proxy doesn't need to manually arbitrate at the connection level — `service_tier` does it natively.

**Critical quota note:** On-demand quota is shared across `priority`, `flex`, and `default` tiers. Setting `priority` does not grant extra quota — it ensures Tier 1 requests are served before Tier 2 when there is contention for the shared quota.

### Surge Queue vs SQS — Clear Boundary

| Scenario | Correct tool |
|---|---|
| Live chatbot user, transient spike, seconds-long wait acceptable | Surge Queue |
| Document processing job throttled, minutes-long wait acceptable | SQS requeue with backoff |
| Bedrock circuit OPEN, sustained outage | SQS requeue with backoff |

---

## 11. Phase 6 — Observability

Observability is not optional. The proxy's correctness depends on real operational data to tune `max_tokens` caps, TPM thresholds, and circuit breaker sensitivity. Without it, the system cannot be tuned and failures are invisible.

### Metrics (CloudWatch Custom Metrics or Prometheus)

| Metric | Purpose |
|---|---|
| Per-tier request count (Tier 1 / Tier 2) | Track traffic split, detect imbalance |
| Circuit breaker state | Track OPEN duration, frequency |
| Surge queue depth | Early warning for sustained pressure |
| TPM consumption vs quota | Track how close to limit in real time |
| `CountTokens` pre-flight results | Understand actual input token distribution per route |
| Requests admitted vs deferred vs shed | Understand admission control effectiveness |

### `ResolvedServiceTier` Tracking

Bedrock exposes both `ServiceTier` (what was requested) and `ResolvedServiceTier` (what actually served the request) in CloudWatch metrics.

**Alert condition:** Divergence between `ServiceTier: priority` and `ResolvedServiceTier: standard` means a Tier 1 chatbot request was downgraded. This indicates sustained quota pressure and is an actionable alert — not just informational.

### Distributed Tracing

- X-Ray or OpenTelemetry across the full proxy hop
- Trace from Lambda caller → proxy → Bedrock → response
- Identify latency spikes, connection pool pressure, and per-route performance

### Alerting

| Alert | Threshold |
|---|---|
| Circuit breaker CLOSED → OPEN | Any transition (immediate alert) |
| Circuit breaker OPEN duration | > 5 minutes (escalate) |
| Surge queue depth | > 80% capacity |
| `ResolvedServiceTier` downgrade | Any priority request downgraded |
| TPM consumption | > 80% of quota for sustained period |
| `max_tokens` cap violations | Callers requesting above per-route ceiling |

---

## 12. Failure & Retry Strategy

### Fail Fast Principle

Worker Lambdas do not retry in-place. When the proxy returns 503 or 429, the Lambda exits immediately. No backoff logic. No sleep. No waiting.

**Why:** A Lambda that retries internally holds a concurrency slot and burns execution time waiting against a quota it is unlikely to win. Fail fast frees the slot immediately.

### SQS Requeue with Exponential Backoff + Jitter

When a worker Lambda fails fast, it requeues the SQS message with a **delay**:

```
delay = min(base × 2^attempt, max_delay) + random(0, jitter_cap)
```

| Parameter | Example Value |
|---|---|
| Base delay | 2 seconds |
| Max delay | 900 seconds (SQS maximum message delay) |
| Jitter cap | 30 seconds |
| Retry attempt | Carried as metadata on the message |

**Why jitter is essential:**

All throttled requests failed at roughly the same time, so without jitter they retry at roughly the same time — recreating the same burst that caused the throttling. Jitter desynchronizes wake-up times across all retrying messages. The retry storm never forms.

### S3 Checkpointing (OCR)

OCR Lambda writes each successfully processed page to S3 before moving to the next.

On retry, the Lambda checks S3 for each page before processing it:
- Page output exists in S3 → skip it
- Page output missing → process it

This makes every retry **idempotent**. Pages 1–25 already in S3 are not reprocessed. The retry resumes from page 26.

Without checkpointing, a 50-page document that fails on page 26 restarts from page 1 on every retry, burning the full OCR token cost repeatedly.

### Step Function Retry Alignment

Step Function retry wait intervals must be aligned with the proxy circuit breaker TTL. If the circuit is OPEN for 60 seconds, Step Function retries should not fire during that window.

**Configuration:** Set Step Function `IntervalSeconds` to at least the circuit breaker TTL. This ensures retries only fire when the circuit has moved to HALF-OPEN and is probing for recovery.

### Lambda SDK Configuration

Disable long exponential backoffs on AWS SDK clients in worker Lambdas:

```python
import boto3
from botocore.config import Config

config = Config(
    retries={
        'max_attempts': 1,      # No SDK-level retries
        'mode': 'standard'
    }
)

bedrock = boto3.client('bedrock-runtime', config=config)
```

SDK-level retries and proxy-level retry logic conflict. The proxy owns retry decisions. The SDK should surface the failure immediately.

---

## 13. Header Contract

All internal services sending requests through the proxy must include:

| Header | Required | Values | Purpose |
|---|---|---|---|
| `Tenant-ID` | Yes | Tenant identifier string | Routes to correct Bedrock AIP ARN |
| `X-Priority` | Yes | `tier-1` or `tier-2` | Determines `service_tier` injected, surge queue position, and eviction priority |
| `Service-ID` | Yes | Service identifier string | Identifies calling service in observability metrics and traces |
| `X-Max-Tokens` | No | Integer | Optional override. Proxy enforces per-route ceiling regardless |

**Proxy behaviour on missing headers:**

| Missing Header | Proxy Action |
|---|---|
| `Tenant-ID` | Return 400. Cannot route without tenant |
| `X-Priority` | Default to Tier 2 (conservative) |
| `Service-ID` | Admit but log missing — observability degraded |

---

## 14. Bedrock AIP & Service Tiers

### Application Inference Profile

Your AIP is the correct target for all real-time inference. Do not target raw model IDs. Targeting the AIP ARN:

- Enables per-tenant cost attribution via resource tags
- Enables cross-region routing to be added later without any proxy code changes
- Provides a stable identifier that survives model version updates

### Service Tier Reference

| Tier | `service_tier` value | Your use case | Characteristic |
|---|---|---|---|
| Reserved | `reserved` | Not yet — future | 99.5% uptime SLA, fixed monthly commitment, separate quota pool. Contact AWS account team |
| Priority | `priority` | Chatbot (Tier 1) | Fastest response, premium price, no reservation needed, served before standard and flex |
| Standard | `default` | General fallback | Default when `service_tier` omitted, balanced performance |
| Flex | `flex` | Document pipeline (Tier 2) | Cost discount, suitable for workloads tolerant of longer processing times |

### TPM Burndown Mechanics (Claude 3.7+)

| Token type | TPM quota deduction |
|---|---|
| Input tokens | 1× (1 token = 1 TPM unit) |
| Output tokens | 5× (1 token = 5 TPM units) |
| `max_tokens` upfront deduction | `max_tokens × 5` at request start |
| Reconciliation | Unused tokens replenished at request end |

**Example:** A request with 1,000 input tokens that generates 100 output tokens:
- TPM deducted: 1,000 + (100 × 5) = 1,500 TPM units
- Billed tokens: 1,100

---

## 15. What Was Deliberately Not Built

| Component | Decision | Reason |
|---|---|---|
| Semaphore Lease API | Removed | The problem it was solving — resumability after mid-document failure — is correctly solved by S3 checkpointing. Lease reservation does not prevent the failure; checkpointing handles recovery cleanly |
| In-memory Surge Queue | Redesigned to Redis-backed | Multiple Fargate tasks run concurrently. In-memory queue state diverges across instances, causing inconsistent priority ordering and potential duplicate dispatch |
| Pure choreography | Rejected | OCR → Summarization → OpenSearch write is a strict sequential dependency. Summarization cannot start without OCR output. QnA and Doc Extraction cannot fire without the summary being indexed. Choreography has no coordinator to enforce this |
| Retry inside Lambda | Rejected | Burns Lambda execution time and concurrency slot waiting. Fail fast + SQS requeue is strictly better — Lambda exits immediately, freeing the slot, and SQS holds the work durably at near-zero cost |
| Batch inference (current phase) | Deferred | 100-record minimum per batch job conflicts with per-upload-event architecture (50 docs per upload). Requires a document accumulation strategy. Viable as a future optimisation once accumulation is designed |
| RPM-based rate limiting at API Gateway | Rejected | LLM calls take 30–90 seconds. RPM is meaningless for requests of this duration. TPM via circuit breaker is the correct governor |

---

## 16. Further Optimisations

These are not Day 1 concerns. Each should be evaluated after the core platform is stable and operational data is available.

### Provisioned Throughput

After 3–6 months of stable operation, CloudWatch TPM metrics will reveal a consistent baseline load. At that point, evaluate purchasing **Model Units (MUs)** via Provisioned Throughput for that baseline.

- MUs provide dedicated capacity — no throttling within purchased units
- Pricing: hourly, with discounts for 1-month and 6-month commitments
- Eliminates throttling for the baseline load entirely
- On-demand (via proxy) handles bursts above the baseline
- Contact AWS account team to initiate

**Decision trigger:** When monthly on-demand Bedrock cost consistently exceeds the equivalent reserved commitment cost.

### Batch Inference

Once a document accumulation strategy is defined that reliably clears the 100-record minimum:

- Document pipeline (OCR + Summarization combined into one record per document) moves to Batch API
- Removes document pipeline traffic from on-demand TPM quota entirely
- 50% cost reduction on that workload
- Frees on-demand quota exclusively for chatbot and real-time interactive traffic

**Accumulation approach options:**
1. Scheduled drain: every N minutes, consume up to M messages from SQS and assemble a JSONL batch job
2. Event-driven accumulation: accumulate across multiple upload events until 100-record threshold is met, then submit

**Note:** Batch inference is a completely separate API from AIP. It does not go through the reverse proxy. Step Functions polls for job completion via EventBridge (Wait for Task Token Callback pattern — no busy polling).

### Reserved Service Tier

For chatbot SLA — if business growth justifies a 99.5% uptime guarantee on Tier 1 traffic:
- Reserved tier provides dedicated capacity separate from on-demand quota pool
- 1-month or 3-month commitment
- Contact AWS account team — not self-serve

### Cross-Region Inference

If a single region's AIP quota becomes a ceiling:
- Create a system-defined cross-region AIP that spans multiple regions
- Swap the AIP ARN in SSM/AppConfig — zero proxy code changes required
- Automatic failover and load distribution across regions

---

*End of document*
