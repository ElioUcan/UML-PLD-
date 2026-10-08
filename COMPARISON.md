# Comparison: Document Database vs. Relational and Key-Value Storage
**Project:** Amenity & Space Reservation System  
**PLD Team:** Nico, Elio  
**Primary Assigned Model:** Document Database (MongoDB)  
**Compared Models:** Relational Storage (PostgreSQL/SQL) & Key-Value Store (Redis)
---
## 1. Architectural & Conceptual Comparison

| Dimension | Primary Model: Document (MongoDB) | Comparison 1: Relational (PostgreSQL / SQL) | Comparison 2: Key-Value (Redis) |
| :--- | :--- | :--- | :--- |
| **Data Representation** | Hierarchical, self-contained BSON documents in collections. Embedded nested objects and arrays. | Normalized 2D tables (rows and columns) connected strictly via primary and foreign key constraints. | Flat key-value pairs stored in-memory (strings, hashes, sets, sorted sets). |
| **Schema Flexibility** | Dynamic / Polymorphic schema. Optional JSON Schema validation per collection. | Rigid, explicit schema. Alterations require DDL migrations (`ALTER TABLE`). | Schemaless. The data structure is governed entirely at application runtime. |
| **Read Characteristics** | **Single-read aggregates:** One document query fetches the reservation, guest list, and check-in status. | **Multi-table joins:** Requires up to 6 joins (`Reservation`, `Guest`, `Payment`, `QRCode`, etc.) to build context. | **Ultra-low latency point lookups ($O(1)$):** Direct key-based fetch; cannot query by nested internal fields. |
| **Write Characteristics** | Atomic updates at the document root level (`$set`, `$push` for adding guests). | Atomic row-level inserts and updates across normalized tables protected by ACID transactions. | In-memory atomic writes and counter increments; persistence is asynchronous or snapshot-based. |
| **Integrity & Constraints** | Application-managed or partial schema validation. Cannot natively constrain time-range overlaps. | Native referential integrity (foreign keys) and spatial/range exclusion constraints (`EXCLUDE USING gist`). | No internal relational constraints; integrity is 100% delegated to client business logic. |
| **Ephemeral Handling** | Built-in TTL (Time-To-Live) indexes; background sweeps clean up expired records periodically. | Manual cleanup jobs (cron/pg_cron) or database trigger-based soft deletes. | Native per-key millisecond precision expiry (`TTL / EXPIRE`) with immediate eviction. |

---
## 2. Fit Analysis for the Reservation System
### When the Document Model (MongoDB) is a GOOD FIT
1. **Aggregated Domain Entity Lifecycle (UC1, UC4):**  
   A reservation is inherently hierarchical. Querying an active booking screen requires the booking details, attached guests, QR pass state, and payment receipt simultaneously. Storing them embedded inside a single reservation document eliminates costly relational `JOIN` operations and keeps the query latency minimal under high read loads.
2. **Dynamic Amenity & Space Attributes:**  
   Different spaces (e.g., BBQ grills, tennis courts, conference rooms) require vastly different configuration attributes, custom rules, and equipment inventories. A document database easily stores heterogeneous JSON structures within the `spaces` collection without leaving empty null-filled columns or requiring complex Entity-Attribute-Value (EAV) relational anti-patterns.
3. **Array Mutation for Guest Lists (UC4):**  
   Appending guests up to maximum capacity is executed using atomic array operations (`$push` with an array-length condition) without locking an external table.
---
### When the Document Model (MongoDB) is a POOR FIT
1. **Time-Slot Overlap Prevention (Concurrency & Booking Conflicts):**  
   MongoDB lacks native range exclusion constraints. Preventing two users from simultaneously booking overlapping intervals for the same space requires distributed locks, complex check-then-insert transactions, or pre-allocated discrete slot bucketing. In contrast, PostgreSQL handles this natively at the engine level using GiST range indexes (`tsrange`).
2. **Financial Ledger & Partial Refund Auditing (UC2):**  
   Payments and penalty-based refund calculations require strict double-entry ledger semantics, audit-trail compliance, and guaranteed cross-entity ACID isolation. Maintaining financial integrity without strict foreign key enforcement introduces data drift risks in high-concurrency environments.
3. **Global Multi-Amenity Analytics & Reporting:**  
   Running complex analytical reports across historical reservations, guest demographics, and refund ratios across multiple spaces requires multi-stage `$lookup` aggregation pipelines, which are significantly less performant and harder to optimize than declarative SQL queries.
---
## 3. Comparative Role of Key-Value Storage (Redis)
While MongoDB handles the aggregated booking entity, **Redis** represents an ideal complementary model rather than a replacement:
* **Good Fit for Access Verification (UC3):**  
  At the physical access scanner / turnstile, throughput and sub-millisecond latency are vital. Storing dynamic QR tokens mapped directly to verification payloads (`qr_token -> {reservation_id, space_id, valid_until}`) enables immediate $O(1)$ lookups and leverages native memory eviction keys once expired.
* **Poor Fit for General Storage:**  
  Redis cannot query inside values or perform range searches over nested customer or schedule attributes. Querying "all confirmed reservations for user X on date Y" would require building and maintaining manual inverted indexes in Redis data structures.
---
## 4. Architectural Synthesis: Polyglot Persistence
Rather than forcing a single storage engine to fulfill contradictory requirements, our reservation system leverages each model for its primary strength:
```mermaid
flowchart TD
    Client[Web / Mobile Client / QR Scanner]
    subgraph Fast Access Layer
        Redis[(Redis - Key-Value)]
    end
    subgraph Operational Domain Layer
        Mongo[(MongoDB - Document)]
    end
    subgraph Transactional & Financial Layer
        Postgres[(PostgreSQL - Relational)]
    end
    Client -->|1. Validate QR Code sub-ms UC3| Redis
    Client -->|2. Manage Bookings, Spaces & Guests UC1, UC4| Mongo
    Client -->|3. Process Payments & Partial Refunds UC2| Postgres
    Mongo -.->|Cache valid tokens with TTL| Redis
    Postgres -.->|Sync transaction reference| Mongo