# Wallet Transfer Service — Approach

## Understanding of the Problem

Build a wallet-to-wallet transfer service that is:

- **Idempotent** — the same request (identified by `idempotencyKey`) must never execute twice, even under retries or duplicate delivery
- **Consistent** — every transfer produces exactly two balanced ledger entries; balances must never drift
- **Concurrency-safe** — concurrent debits on the same wallet must not cause double-spending or negative balances
- **Auditable** — transfer state transitions are persisted and queryable

---

## Schema Design

### `wallets`

```sql
CREATE TABLE wallets (
  id          TEXT        PRIMARY KEY,
  name        TEXT        NOT NULL,
  balance     NUMERIC(18,2) NOT NULL DEFAULT 0,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

`balance` is a cached running total updated inside the same transaction as every ledger write.
This gives O(1) balance reads. Correctness is verifiable at any time by comparing
`SUM(ledger_entries.amount WHERE wallet_id = X)` against `wallets.balance`.

### `transfers`

```sql
CREATE TABLE transfers (
  id               UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  idempotency_key  TEXT        UNIQUE NOT NULL,
  from_wallet_id   TEXT        NOT NULL REFERENCES wallets(id),
  to_wallet_id     TEXT        NOT NULL REFERENCES wallets(id),
  amount           NUMERIC(18,2) NOT NULL CHECK (amount > 0),
  status           TEXT        NOT NULL DEFAULT 'PENDING'
                               CHECK (status IN ('PENDING','PROCESSED','FAILED')),
  failure_reason   TEXT,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

`idempotency_key` has a `UNIQUE` constraint directly on the transfers table.
This is the database-level guarantee that no two transfers share the same key.

### `ledger_entries`

```sql
CREATE TABLE ledger_entries (
  id           UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  transfer_id  UUID        NOT NULL REFERENCES transfers(id),
  wallet_id    TEXT        NOT NULL REFERENCES wallets(id),
  type         TEXT        NOT NULL CHECK (type IN ('DEBIT','CREDIT')),
  amount       NUMERIC(18,2) NOT NULL CHECK (amount > 0),
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Every transfer produces exactly two rows: one DEBIT from the source wallet, one CREDIT to the destination wallet.
Amount is always stored as a positive number; the `type` column encodes direction.

### Why no separate `idempotency_records` table

The `UNIQUE` constraint on `transfers.idempotency_key` is sufficient.
A separate idempotency table would require a two-phase write (insert idempotency record, then insert transfer)
with its own race conditions. Colocating the key on the transfer row means the
idempotency guarantee and the business record are created atomically in one INSERT.

---

## Idempotency Strategy

### How the key is stored

`idempotency_key` is a column on the `transfers` table with a `UNIQUE` constraint.

### How duplicate requests are detected — three layers

**Layer 1 — Fast path (no lock):**
Before any transaction, query for an existing transfer with the same key.
If found, return the cached result immediately without touching the DB further.

**Layer 2 — Guarded INSERT:**
Inside the transaction, attempt `INSERT INTO transfers (idempotency_key, ...)`.
If the key already exists, Postgres raises a unique violation.
The handler catches this, fetches the existing transfer, and returns it.

**Layer 3 — Unique constraint:**
Two concurrent requests with the same key can both pass Layer 1 (neither has written yet).
Both attempt the INSERT. Postgres serialises them at the constraint: exactly one succeeds,
the other receives a constraint violation and falls through to return the existing record.

Layer 1 reduces unnecessary DB round-trips. Layer 3 is the correctness guarantee.

### How duplicate side effects are prevented

Because the `idempotency_key` INSERT and all business writes (ledger entries, balance updates)
happen inside a single transaction, a duplicate request that loses the constraint race
never sees any partial state — the transaction that won either commits fully or rolls back fully.

---

## Concurrency Strategy

### Approach: pessimistic row-level locking (`SELECT FOR UPDATE`)

When a transfer executes:

1. Sort the two wallet IDs lexicographically (lower ID first)
2. `SELECT ... FROM wallets WHERE id IN ($1, $2) ORDER BY id FOR UPDATE`
3. Read balances from the locked rows
4. Check that `from_wallet` has sufficient funds
5. Write ledger entries and update balances
6. Commit

### Why lock ordering prevents deadlocks

Without consistent ordering, two concurrent transfers in opposite directions deadlock:

```
T1: wallet_A → wallet_B  — locks A, waits for B
T2: wallet_B → wallet_A  — locks B, waits for A
Both wait forever — DEADLOCK
```

Sorting by wallet ID before locking means both transactions always acquire locks in the same order,
so the second one simply waits for the first to commit. No deadlock, no retry needed.

### Why pessimistic locking over optimistic locking

Optimistic locking (version columns + retry on conflict) introduces retry loops in the application layer.
Under high contention, retries can cascade, making latency unpredictable.
Pessimistic locking with consistent ordering is more explicit and gives predictable,
deadlock-free behaviour with no retry complexity — the right default for a financial system
where correctness matters more than throughput.

---

## Transfer State Machine

```
PENDING → PROCESSED   (funds available, ledger written, balances updated)
PENDING → FAILED      (insufficient funds)
```

**Why PENDING exists:**

The transfer row is inserted with `status = PENDING` before business logic executes.
This means:

- If the process crashes mid-execution, a PENDING row is left behind as evidence
- A background reconciliation job can detect stale PENDING transfers and mark them FAILED
- The idempotency_key is claimed the moment the row is inserted, preventing any concurrent
  duplicate from creating a second row even if the first is still in PENDING

**State transitions are terminal:**
A PROCESSED or FAILED transfer cannot transition to any other state.
Reversal is a separate transfer, not a state change.

---

## API Contract

### POST /transfers

**Request:**
```json
{
  "idempotencyKey": "abc123",
  "fromWalletId": "wallet_1",
  "toWalletId": "wallet_2",
  "amount": "100.00"
}
```

`amount` is sent as a string to avoid float64 precision loss at the JSON boundary.
Parsed with exact decimal arithmetic on the server.

**Response (201 Created or 200 OK on duplicate key):**
```json
{
  "id": "uuid",
  "idempotencyKey": "abc123",
  "fromWalletId": "wallet_1",
  "toWalletId": "wallet_2",
  "amount": "100.00",
  "status": "PROCESSED",
  "createdAt": "2026-05-17T10:00:00Z"
}
```

**Error responses:**
- `400` — malformed request (missing fields, non-positive amount, same source and destination)
- `409` — idempotency key conflict with different request body (same key, different wallets/amount)
- `422` — insufficient funds
- `404` — wallet not found

**Duplicate key behaviour:**
- Same key + same body → `200 OK` with original response
- Same key + different body → `409 Conflict`

### GET /wallets/:id/balance *(optional)*

Returns current balance derived from cached `wallets.balance`.

### GET /transfers/:id *(optional)*

Returns a single transfer with its ledger entries.

---

## Failure Modes and Handling

| Scenario | Behaviour |
|---|---|
| Insufficient funds | INSERT transfer(FAILED, failure_reason), no ledger entries, COMMIT |
| Same source and destination | Rejected at validation, no DB write |
| Non-positive amount | Rejected at validation, no DB write |
| Duplicate idempotency key (same body) | Return original result, no second transfer |
| Duplicate idempotency key (different body) | 409 Conflict |
| Process crash mid-transfer | PENDING row left; balance unchanged (no ledger written yet); safe to retry |
| Wallet not found | 404, no DB write |

**Why COMMIT on failure (insufficient funds):**
A FAILED transfer row is committed, not rolled back.
This preserves the evidence that the attempt happened, which is essential for auditability.
The audit trail must be complete even for failed attempts.

---

## Testing Strategy

### Unit tests

- Service layer: transfer execution, idempotency check logic, state transitions
- Domain validation: negative amounts, same-wallet transfers, missing fields

### Integration tests (against real Postgres)

- Happy path: transfer executes, two ledger entries created, balances updated correctly
- Insufficient funds: transfer marked FAILED, no ledger entries, balances unchanged
- Idempotency: same key sent twice returns identical response, only one transfer row in DB
- Conflicting idempotency key: same key, different body → 409
- Ledger balance invariant: `SUM(ledger_entries.amount WHERE type=CREDIT) = SUM(... WHERE type=DEBIT)` across all entries for a transfer
- Ledger ↔ balance consistency: `SUM(ledger_entries.amount per wallet)` matches `wallets.balance`

### Concurrency test

- N goroutines concurrently transferring between the same two wallets
- Assert: no negative balances, money is conserved (total balance across wallets unchanged), ledger matches balance
- Same idempotency key sent by M goroutines concurrently
- Assert: exactly one transfer row exists, all M responses return the same transfer ID

---

## Assumptions and Trade-offs

| Decision | Rationale |
|---|---|
| `idempotency_key` on `transfers` table, not a separate table | Atomicity — key is claimed and transfer is created in one INSERT. Separate table needs two writes with its own race. |
| Cached balance on `wallets` | O(1) reads; correctness verifiable by comparing to ledger SUM at any time |
| Pessimistic locking | Predictable latency under contention; no retry loops |
| Lock ordering by wallet ID | Prevents deadlocks without needing serializable isolation |
| Amount as string in API | Avoids float64 precision loss at the JSON boundary |
| FAILED transfers are committed (not rolled back) | Audit trail must include failed attempts |
| No authentication | Out of scope for this assignment |
| No reversal API | Not in functional requirements; can be added as a subsequent transfer if needed |

---

## Implementation Plan

1. Schema + migrations
2. Domain models and error sentinels
3. Repository layer (wallet reads with FOR UPDATE, transfer INSERT, ledger INSERT, balance UPDATE)
4. Service layer (idempotency check, lock ordering, business logic, state transitions)
5. Handler layer (request parsing, validation, error mapping)
6. Integration tests
7. Concurrency test
