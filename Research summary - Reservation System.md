# Research Summary: Document Databases (MongoDB) for a Reservation System

Polyglot Persistence PLD. Team: Nico, Elio. Model assigned: document database. Example domain: space reservation system (booking, cancellation with partial refund, QR check-in, guest registration).

## 1. What a document database is

A document database stores data as self-contained, JSON-like documents instead of rows spread across tables. MongoDB stores them in BSON, a binary JSON format with extra types such as dates, decimals and ObjectIds. Documents live in **collections**, which play the role of tables but do not force every document to have the same fields.

It exists because many applications read and write one logical thing at a time (an order, a profile, a booking) and that thing is naturally nested. In SQL it is split across several tables and rebuilt with joins. A document database lets you keep it in one place, evolve the shape without migrations, and scale out across servers more easily.

## 2. How data is represented

Core building blocks:

- **Document**: one record, a set of key/value pairs. Values can be strings, numbers, dates, arrays, or other documents (nesting).
- **Collection**: a group of related documents.
- **\_id**: the unique primary key of every document, generated automatically if you do not supply it.
- **Embedding vs referencing**: the main design decision. Embed data that is read together and owned by the parent; reference (store an id) data that is shared or grows without bound.

Applied to the reservation domain, the UML entities Reservation, Payment, Refund, QRCode, CheckIn and Guest can all be embedded in a single reservation document, while User and Space stay in their own collections and are referenced by id:

```json
{
  "_id": "res_1001",
  "user_id": "usr_42",
  "space_id": "spc_terrace",
  "start_time": "2026-10-20T18:00:00Z",
  "end_time": "2026-10-20T22:00:00Z",
  "status": "CONFIRMED",
  "total_amount": 800,
  "payment": { "amount": 800, "method": "card", "status": "PAID" },
  "qr_code": { "token": "a9f3...", "status": "VALID", "expiration_time": "2026-10-20T22:00:00Z" },
  "guests": [
    { "full_name": "Ana Lopez", "id_document": "INE-123", "access_status": "PENDING" }
  ],
  "refund": null,
  "check_in": null
}
```

In SQL this same booking touches up to six tables. Here one read returns everything the booking screen needs.

## 3. Basic operations

With the official driver (for example PyMongo) or the shell:

- **Create**: `insertOne` / `insertMany` add documents.
- **Read**: `find(filter)` supports comparison operators, nested paths (`"payment.status"`) and array matching.
- **Update**: `updateOne` with operators such as `$set`, `$push` (add a guest) and `$inc`.
- **Delete**: `deleteOne` / `deleteMany`.

Mapped to the use cases: UC1 inserts a reservation, UC4 pushes onto `guests`, UC3 sets `status` to CHECKED\_IN and fills `check_in`, UC2 sets `status` to CANCELLED and fills `refund`.

## 4. Distinguishing features to demo

- **Nested documents and arrays**: query guests inside a reservation directly, with no join (`find({"guests.full_name": "Ana Lopez"})`).
- **TTL index**: a time-to-live index makes the database delete documents automatically after a date. A natural fit for expiring QR passes or pending reservations that were never paid. Note that the TTL cleanup runs periodically, so expiry is not instantaneous; the check-in logic should still verify `expiration_time`.
- **Schema validation**: optional JSON Schema rules per collection, so flexibility can be tightened where needed (for example `status` must be one of four values).
- **Multi-document transactions**: supported on replica sets, useful for atomically cancelling a reservation and releasing its slot. Embedding reduces the need for them.

## 5. Strengths and trade-offs

**Strengths**

- Flexible schema: new fields (for example a guest email) need no migration.
- Data that is read together is stored together, which gives simple, fast reads.
- Natural mapping to application objects and JSON APIs.
- Built-in horizontal scaling (sharding) and replication.

**Trade-offs**

- No enforced foreign keys: nothing stops a reservation pointing to a space that does not exist unless the application checks.
- Duplication and consistency risk when the same data is embedded in many places.
- Cross-collection queries and reporting (for example revenue per space per month across users) are more awkward than SQL joins, though the aggregation pipeline and `$lookup` help.
- Preventing overlapping bookings is hard to express as a constraint. A unique index can block two identical slots, but not two overlapping time ranges, so the app needs a check-then-insert inside a transaction, or a design based on fixed time slots.
- Document size limit (16 MB) and the need to think about unbounded arrays.

## 6. Comparison

**Versus relational (SQL).** Simpler in a document store: loading a full reservation, adding fields, storing varied space types with different attributes. Harder: enforcing integrity (foreign keys, no overlapping bookings), ad hoc reporting across entities, and multi-entity transactional guarantees. Payments and refunds are the strongest argument for staying relational, because money needs strict consistency and auditability.

**Versus key-value (for example Redis).** Key-value stores are faster for simple lookups but cannot query inside values. Redis would suit the QR token lookup at the door (token to reservation, with TTL), while MongoDB suits the reservation itself, which needs filtering by date, status or guest.

**Good fit** when the data is hierarchical, read as a unit, and the schema changes often. **Poor fit** when relationships and constraints are central or when complex multi-table analytics dominate.

## 7. Where it fits in a polyglot architecture

For this system: PostgreSQL (or similar) for payments, refunds and the financial ledger; MongoDB for reservations, guests and space catalog with differing attributes; Redis for short-lived QR token validation and caching availability. Each store handles what it does best, at the price of more operational complexity and the need to keep them in sync.

## 8. Proof of concept plan

Run MongoDB from the supplied Docker image and connect with a client library. Script: insert spaces and a reservation, query by nested field, update (add guest, check in, cancel with refund), delete, then demo the TTL index on QR codes. Keep the code short, commented and runnable with a README.

## 9. Sources and open items

Sources to cite (verify and add the course's curated resources):

- MongoDB Manual, official documentation: https://www.mongodb.com/docs/manual/ (CRUD operations, data modeling, TTL indexes, schema validation, transactions).
- PyMongo documentation: https://pymongo.readthedocs.io/
- The team's UML design (`README.md`, `INFRA.md`) for the entities and use cases.

To complete as a team: add anything that surprised you during research, and note any conflicting information you found, since the brief rewards pointing out caveats. Also confirm the exact collection design after running your POC, because the embedded layout above is a proposal, not a tested result.
