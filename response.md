# Challenge  
**Candidate Assessment – 30 min Interview**

## Context  
In this exercise, we want to see how you design and reason about scalable systems — not a perfect implementation.  
Please focus on clarity, trade-offs, and communication.  
You will have 30 minutes.  
We expect a lightweight deliverable (diagram + notes + small code/SQL snippets).  
No need to run code or install tools.  

---

## The Task  
Design a high-volume event-driven metrics platform.

### Requirements  
- The system must ingest up to 10k events per second (user actions, e.g., “user clicked button”).  
- Events must be idempotent (no duplicates).  
- Ordering must be preserved per user.  
- We need to query top-N most active users over a time window (e.g., last 24 hours).  
- Results must be exposed via a REST API.  
- The system should be cost-efficient and resilient to failures.  

---

## Deliverable  
Please prepare, in any format you like (Markdown, Google Doc, Miro/Draw.io, or even hand-drawn and explained):  

1. **Architecture flow** – a diagram or bullet list of the components (producers, queues, processors, storage, API, etc.).  
2. **Key design decisions** – what technologies/patterns you would choose and why (e.g., SQL vs NoSQL, caching, batching, stream processing).  
3. **Trade-offs** – discuss latency vs cost, consistency vs availability, etc.  
4. **Failure handling** – how to avoid duplicates, handle retries, and process errors.  
5. **Examples (short snippets)** –  
   - An example SQL query to get “top-N users by event count.”  
   - An example API endpoint (URL + method) for fetching the metrics.  

---

# Response  

## Main Decisions  
- **API (NodeJS or Go):** lightweight containers with very fast languages to process records as fast as possible and send them to Kafka, behind an ALB for load balancing.  
- **Ordering:** Kafka will use the `user_id` as the partition key, ensuring order is preserved per user.  
- **Idempotency:** All producers and consumers will run `acks=all`, making events idempotent.  
- **Database choice:** ClickHouse for handling top-N queries over large datasets cost-effectively. Alternatives like Redshift are possible but ClickHouse is optimized for this use case.  
- **REST API:** `/events/` (GET) endpoint that returns events with optional query parameters.  
- **Cost efficiency & resilience:**  
  - S3 for replayability at low cost.  
  - ALB instead of API Gateway to save costs.  
  - Node API in lightweight ECS containers for great performance.  
  - Potential use of serverless if load is inconsistent.  

---

## Trade-offs  

- **Latency vs Cost:**  
  - Redis for sub-10ms reads (higher cost/complexity).  
  - Direct ClickHouse queries (~50–300ms, cheaper).  

- **Consistency vs Availability:**  
  - If Redis is down, fallback to ClickHouse queries.  

- **Exactly-once vs Complexity:**  
  - EOS and deduplication at the stream level offloads complexity from consumers.  

---

## Failure Handling  

- Raw event storage in S3 for replayability.  
- Events uniquely identified via UUID.  
- Producers implement backoff/retry strategies.  
- Kafka ensures durability and supports event reprocessing.  
- Idempotent upserts on S3 mitigate duplicates.  

---

## Query Samples  

### Raw Events Table  

```sql
CREATE TABLE events_raw (
  event_date Date DEFAULT toDate(ts),
  ts DateTime64(3, 'UTC'),
  event_id String,
  user_id String,
  type LowCardinality(String),
  props JSON
) ENGINE = MergeTree
PARTITION BY toYYYYMM(event_date)
ORDER BY (user_id, ts);
