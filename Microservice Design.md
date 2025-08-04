# Question 1 — Microservice Design

## 1. Overview
The goal of this microservice is to provide endpoints for reading and writing data in a scalable way, keeping latency under *500ms* even when there is millions or billions of records and many concurrent users.

The design follow *Clean Architecture* and *SOLID* principles, with clear separation of responsabilities and well defined layers.

---

## 2. Internal Architecture

### 2.1 API Layer
•⁠  ⁠*Responsibility:* Receive and validate HTTP requests, call the service layer and format responses.
•⁠  ⁠*Suggested technologies:*  
  - *FastAPI* (Python) or *NestJS* (Node.js)
  - OpenAPI/Swagger for automatic documentation
•⁠  ⁠*Best practices:*
  - Input validation (Pydantic, Joi, class-validator)
  - Authentication/authorization (JWT, OAuth 2.0)
  - Basic rate limiting for abuse protection

---

### 2.2 Service Layer
•⁠  ⁠*Responsibility:* Encapsulate business rules and decide how and where the data will be read/persisted.
•⁠  ⁠*Main functions:*
  - Determine the correct *shard* based on insertion date (⁠ timestamp ⁠)
  - Apply extra validations and data consistency rules
  - Decide if it will use cache or database directly depending on the query

---

### 2.3 Repository Layer
•⁠  ⁠*Responsibility:* Isolate data persistence and access logic.
•⁠  ⁠*Components:*
  - *PostgreSQL* with *temporal sharding* (monthly/quarterly) for better scalability
  - *Redis Cluster* for cache of the hottest data (hot shards)
•⁠  ⁠*Benefits:*
  - Can change database or cache without affecting other layers
  - Easier to maintain and evolve

---

## 3. Persistence Strategy

### PostgreSQL Sharding
•⁠  ⁠*Criteria:* Data insertion date (⁠ timestamp ⁠)
•⁠  ⁠*Hot shards:* Recent data (high access frequency)
•⁠  ⁠*Cold shards:* Old data (can be in cheaper instances)
•⁠  ⁠*Benefits:*
  - Queries target less shards → lower latency
  - Easy to archive or delete old data
  - Possible to scale horizontally

### Redis Cluster (HA) for Hot Shards
•⁠  ⁠*Setup:*
  - Cluster mode with 3 primary nodes + 3 replicas
  - Automatic failover and high availability
•⁠  ⁠*Usage:*
  - Cache frequent reads to reduce latency to 1–5ms
  - TTL configured to avoid stale data
•⁠  ⁠*Benefits:*
  - Resilience against failures
  - Consistent performance

---

## 4. Operation Flows

### POST /data
1.⁠ ⁠*API Layer*:
   - Receive JSON
   - Validate required fields (⁠ userId ⁠, ⁠ timestamp ⁠, ⁠ payload ⁠)
   - Return ⁠ 422 ⁠ if invalid
2.⁠ ⁠*Service Layer*:
   - Determine shard by ⁠ timestamp ⁠
   - Apply business rules
3.⁠ ⁠*Repository Layer*:
   - Insert in the correct shard (PostgreSQL)
   - Optional: insert into Redis if is hot data
4.⁠ ⁠*Response*:
   - ⁠ 201 Created ⁠ with the ID of the record

---

### GET /data
1.⁠ ⁠*API Layer*:
   - Receive filters (⁠ userId ⁠, ⁠ date range ⁠)
   - Validate params
2.⁠ ⁠*Service Layer*:
   - Determine relevant shards
   - If hot data → try reading from Redis
   - If cache miss → query PostgreSQL
3.⁠ ⁠*Repository Layer*:
   - Fetch data
   - Populate cache if needed
4.⁠ ⁠*Response*:
   - ⁠ 200 OK ⁠ with JSON results

---

## 5. Internal Diagram

```plaintext
          Client
            │
        API Layer
 (Validation, Auth, Docs)
            │
        Service Layer
 (Business rules, shard)
            │
     Repository Layer
  ┌──────────┴──────────┐
  │                     │
Redis Cluster        PostgreSQL
 (Hot Cache)        (Sharded DB)

---

## 6. Final Notes
  • Decoupled architecture makes maintenance and testing easier.
  • Temporal sharding simplify scalability and data management.
  • Redis Cluster ensure high availability and low latency for hot data.
  • Design can evolve to async messaging (Kafka, RabbitMQ) if needed.
