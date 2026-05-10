---
simd: 'XXXX'
title: Constellation Replay Stage
authors:
  - (fill in with names of authors)
category: Standard
type: Core
status: Idea
created: 2026-05-11
feature: (fill in with feature key and github tracking issues once accepted)
extends: SIMD-XXXX, SIMD-XXXX, SIMD-XXXX
---

## Summary

This proposal defines the Constellation replay stage. Replay validators
deterministically turn leader-included proposal submissions into candidate
transactions, apply replay-time transaction filters, charge inclusion and
execution fees, compute a deterministic execution schedule, execute scheduled
transactions with Solana SVM semantics, update Constellation replay state, and
freeze the resulting Bank.

Given the same parent Bank and Constellation block payload, validators compute
the same candidate set, fee deductions, inclusion-only results, execution
schedule, Bank state, and Bank hash.

## New Terminology

A `candidate transaction` is a unique transaction produced by flattening all
leader-included submissions in a batch.

An `included-by set` is the set of proposal references that included the same
candidate transaction.

An `inclusion-only result` records that a transaction paid inclusion fees but was
not executed because it could not pay execution fees while preserving the
reserve floor.

An `execution lane` is one of `m` deterministic scheduling lanes used to assign
non-conflicting transactions.

A `batch makespan` is the maximum scheduled end CU across all lanes for a batch,
or zero if no transactions are scheduled.

`Inclusion reserve state` is Bank state used to enforce the reserve floor and
pending reserve changes referenced by proposer and replay rules.

## Detailed Design

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in [RFC
2119](https://www.ietf.org/rfc/rfc2119.txt) and [RFC
8174](https://www.ietf.org/rfc/rfc8174.txt).

This proposal depends on SIMD-XXXX for common parameters, SIMD-XXXX for
transaction and proposal-slice semantics, SIMD-XXXX for block and batch payload
validity, and SIMD-XXXX for committing the replay result with the block footer
`bank_hash`.

### Replay input

For each batch, replay receives:

```
struct ReplayBatchInput {
    parent_bank: Bank,
    batch: ConstellationBatch,
    prior_attested_refs: BTreeSet<ProposalRef>,
    status_cache: StatusCache,
}
```

### Flatten submissions

Replay starts by flattening all transactions in all submissions.

Each candidate records which proposers included it:

```
struct CandidateTx {
    tx: VersionedTransaction,
    tx_id: Signature,              // first signature
    message_hash: Hash,
    tx_hash: Hash,

    priority_bid: u64,             // from ComputeBudget::SetComputeUnitPrice
    cu_limit: u64,                 // effective requested CU limit
    inclusion_bid: u64,            // lamports per byte
    byte_len: u64,

    writable_accounts: BTreeSet<Pubkey>,

    included_by: BTreeSet<ProposalRef>,
}
```

Flattening is deterministic:

```
candidates = empty map keyed by tx_id/message_hash

for submission in batch.submissions in canonical order:
    for tx in submission.transactions:
        sanitize tx
        tx_id = first signature
        message_hash = hash(message bytes)

        if candidates does not contain (tx_id, message_hash):
            insert CandidateTx(tx)

        add submission.proposal_ref to candidate.included_by
```

If the same transaction ID appears with different message bytes, the block is
invalid.

If a transaction is already in the Bank's status cache as processed, the block
is invalid. The proposer stage rejects processed transactions against its base
Bank, and replay treats a processed included transaction as invalid proposer
output.

### Candidate replay filters

Before deterministic assignment, each candidate MUST pass:

- transaction parses and sanitizes
- signatures verify
- address lookup tables resolve
- compute-budget instructions are valid
- effective `cu_limit <= tcu`
- `recent_blockhash` is valid or durable nonce is valid
- `expire_cycle >= batch.cycle`
- fee payer is a valid Solana fee payer
- `target_proposer_mask` includes every proposer index in `included_by`

A candidate failing these checks makes the block invalid if it was inside an
included proposal slice.

### Fee definitions

For candidate `tx`:

```
inclusion_fee_per_ref(tx) = tx.byte_len * tx.inclusion_bid
total_inclusion_fee(tx)  = inclusion_fee_per_ref(tx) * |included_by|

base_fee(tx)             = Solana signature/base fee under the Bank's fee schedule
priority_fee(tx)         = ceil(priority_bid_micro_lamports_per_cu * cu_limit / 1_000_000)
execution_fee(tx)        = base_fee(tx) + priority_fee(tx)
```

Replay charges fees as follows:

- every candidate in an included proposal slice: charge inclusion fees
- execution fee payment would violate the reserve floor: charge inclusion fee
  only
- skipped due to capacity or conflicts: charge inclusion fees only
- selected for execution: charge execution fee in addition to inclusion fees
- execution fails: charged fees remain committed

Executed transactions retain charged fees even if execution fails.

### Fee-payer reserve rule during replay

Let:

```
fp = fee payer account
balance(fp) = current lamports in replay Bank/cache
reserve_floor = phi
```

Execution eligibility check:

```
balance(fp) - execution_fee(tx) >= reserve_floor
```

Inclusion fees are charged before the execution eligibility check. If false,
replay:

1. records status as `InclusionOnly` or `FeesOnly`
2. skips transaction instruction execution
3. continues to the next candidate

If true, the transaction may be scheduled. Once actually scheduled, replay
deducts `execution_fee(tx)`.

Inclusion fees are paid from the inclusion reserve state and MAY reduce the
account below `phi`. Program execution itself MUST preserve the reserve floor.
If instruction effects would reduce the fee payer below the reserve rule, the
transaction fails and only fees remain committed.

If an inclusion-fee debit would underflow lamports, the block is invalid.

### Deterministic transaction ordering

Candidate transactions are sorted by:

1. higher priority bid first
2. lower CU limit first
3. lower transaction hash first
4. lower transaction ID first, as final deterministic tie-break

Formally:

```
sort_key(tx) = (
    Reverse(tx.priority_bid),
    tx.cu_limit,
    tx.tx_hash,
    tx.tx_id,
)
```

### Deterministic transaction assignment

The assignment algorithm returns a schedule:

```
struct ScheduleEntry {
    tx_id: Signature,
    lane: u8,          // 0 <= lane < m
    start_cu: u64,
    end_cu: u64,
}
```

Two transactions conflict if their writable account sets intersect:

```
conflict(tx_a, tx_b) =
    tx_a.writable_accounts intersect tx_b.writable_accounts != empty
```

Read-only overlap is schedulable.

Assignment state:

```
free_intervals[lane] = initially [(0, mcu)] for every lane
placed = []
```

Each placed transaction has:

```
(tx_id, lane, start_cu, end_cu, writable_accounts)
```

For each transaction `tx` in sorted order:

- charge `total_inclusion_fee(tx)`
- if `tx` fails the reserve-floor execution-fee check, skip scheduling
- otherwise, find the earliest feasible placement across all lanes
- if no feasible placement exists, skip scheduling
- if feasible, place it, charge execution fee, and update free intervals

Placement procedure:

```
best = None

for lane in 0..m-1:
    for interval (a,b) in free_intervals[lane] sorted by (a,b):
        s = a

        while s + tx.cu_limit <= b:
            conflicting = all placed y such that:
                y.start_cu < s + tx.cu_limit
                and s < y.end_cu
                and conflict(tx, y)

            if conflicting is empty:
                candidate = (s, lane)
                best = min(best, candidate) by (start_cu, lane)
                break

            s = min(y.end_cu for y in conflicting)

if best is None:
    tx is not scheduled
else:
    place tx at best
```

After placement at `(s, lane)`:

```
end = s + tx.cu_limit
split the chosen free interval:
    (a,b) -> optionally (a,s) and optionally (end,b)
insert ScheduleEntry(tx, lane, s, end)
```

The batch makespan is:

```
ms(batch) = max(entry.end_cu for entry in schedule), or 0
```

The batch is valid only if:

```
ms(batch) <= mcu
```

The block is valid only if:

```
sum(ms(batch_i)) <= bcu
```

### Execution order and parallel replay

A validator MAY execute transactions in parallel, but the resulting state MUST
be equivalent to execution in the canonical order:

```
(start_cu, lane, tx_hash, tx_id)
```

Because the schedule forbids overlapping writable-account conflicts, concurrent
transactions have disjoint writable account sets. For conflicting transactions
placed at non-overlapping times, the earlier interval is the predecessor.

Execution uses Solana SVM semantics:

- load accounts
- validate nonce if applicable
- execute instructions sequentially inside the transaction
- if any instruction fails, roll back non-fee state changes
- commit fee deductions and nonce advancement as required
- update status cache

Constellation changes candidate selection, assignment, and fee timing, not SVM
instruction semantics.

### Batch replay procedure

Replay for a single batch is:

```
ReplayBatch(bank, batch):

    ValidateBatchMetadata(batch)

    candidates = FlattenSubmissions(batch.submissions)

    candidates = ValidateAndBuildCandidateSet(bank, candidates)

    ordered = sort(candidates, sort_key)

    schedule = []

    for tx in ordered:
        if status_cache contains tx:
            reject block

        charge_inclusion_fee(bank, tx)

        if !can_pay_execution_fee_preserving_reserve(bank, tx):
            status_cache.insert(tx, InclusionOnly)
            continue

        placement = first_fit(tx, schedule)

        if placement is None:
            continue

        charge_execution_fee(bank, tx)

        schedule.insert(placement)

    ExecuteScheduledTransactions(bank, schedule)

    bank.highest_constellation_cycle = batch.cycle

    return bank
```

### Block replay procedure

Replay for a Constellation block is:

```
ReplayConstellationBlock(parent_bank, payload):

    CheckBlockHeader(payload, parent_bank)

    prev_attested = parent_bank.constellation_attested_refs
    bank = parent_bank.new_child_bank(payload.slot)

    total_makespan = 0
    expected_cycle = ComputeFirstExpectedCycle(parent_bank, payload.slot)

    for batch in payload.batches:
        CheckCycleValidity(batch, expected_cycle)

        CheckBatchAttestationsAndSubmissions(batch, prev_attested)

        schedule_preview = DeterministicAssignmentPreview(bank, batch)
        total_makespan += makespan(schedule_preview)

        if total_makespan > bcu:
            reject block

        bank = ReplayBatch(bank, batch)

        prev_attested += attested_refs(batch)
        expected_cycle = batch.cycle + 1

    bank.constellation_attested_refs = prev_attested
    bank.freeze()

    return bank
```

The resulting Bank hash is committed by the block footer `bank_hash` described
by SIMD-XXXX. A mismatch between the replayed Bank hash and the block footer
bank hash makes the block invalid.

