---
simd: 'XXXX'
title: Constellation Consensus Leader and Voter Validity
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

This proposal defines how the scheduled Solana slot leader constructs
Constellation batches and block payloads from attestation bundles and forwarded
proposal shreds. It also defines the Constellation-specific pre-vote validity
checks that validators apply before voting for a slot.

## New Terminology

A `Constellation batch` is the replayable data for one attestation cycle inside
a block.

A `submission` is the leader-included transaction list for a newly attested
proposal reference.

A `proposal reference` identifies a proposal slice by epoch, proposal cycle,
proposer index, proposal slice index, and transaction-list hash.

## Detailed Design

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in [RFC
2119](https://www.ietf.org/rfc/rfc2119.txt) and [RFC
8174](https://www.ietf.org/rfc/rfc8174.txt).

This proposal depends on SIMD-XXXX for schedules and common parameters,
SIMD-XXXX for proposal slices and proposal shreds, and SIMD-XXXX for
attestation validity.

### Leader role

The consensus leader is the scheduled Solana slot leader. It receives
attestation bundles and encrypted proposal shreds from attesters, constructs
Constellation batches, inserts them into the block payload, serializes entries,
and broadcasts normal Solana ledger shreds.

### Batch format

A batch corresponds to one attestation cycle:

```
struct ConstellationBatch {
    epoch: Epoch,
    cycle: u64,

    // Exactly alpha bits set.
    attester_indices: BitVec<q>,

    // One per selected attester, sorted by attester_index.
    attestations: Vec<AttestationPlus>,

    // One per newly attested valid proposal slice, in canonical order.
    submissions: Vec<Submission>,
}

struct AttestationPlus {
    epoch: Epoch,
    attestation_cycle: u64,
    attester_index: u16,
    lists: Vec<AttestationList>, // length p
    signature: Signature,
}

struct Submission {
    proposal_ref: ProposalRef,
    transactions: Vec<VersionedTransaction>,
}

struct ProposalRef {
    epoch: Epoch,
    proposal_cycle: u64,
    proposer_index: u16,
    pslice_index: u64,
    tx_list_hash: Hash,
}
```

`AttestationPlus` preserves the original attester signature.

### Block payload format

The slot leader creates a Constellation block payload:

```
struct ConstellationBlockPayload {
    version: u16,
    slot: Slot,
    parent_slot: Slot,
    parent_bank_hash: Hash,
    leader_identity: Pubkey,

    batches: Vec<ConstellationBatch>,
}
```

### Valid attestation set for a batch

For batch cycle `c`, the leader selects a set `Q` of exactly `alpha = q / 2`
unique attester indices.

All selected attestations MUST be valid for `(epoch, c)`:

```
|Q| = alpha
all k in Q are unique
for each k in Q, attester(epoch,c)[k] signed a valid attestation
```

The leader MAY have received more than `alpha` valid attestations. Block
validity only checks that the included `Q` is valid.

### Aggregate attestation threshold

For a selected attestation set `A`, define:

```
count(ref) = number of attestations in A whose full list contains ref
```

A proposal reference is attested in `A` iff:

```
count(ref) >= gamma_p
```

All included attestation lists are full lists and count toward commitments.

### Newly attested references

Let `prev_attested` be the set of proposal references attested in all ancestor
blocks and earlier batches in the current block.

For batch `b`:

```
newly_attested(b) =
    sorted(attested(b.attestations) - prev_attested)
```

The canonical sort order is:

```
(epoch, proposal_cycle, proposer_index, pslice_index, tx_list_hash)
```

The batch MUST contain exactly one `Submission` for each newly attested proposal
reference.

### Proposal reconstruction

For each newly attested `ProposalRef ref`, the leader reconstructs the proposal
slice.

First, the leader collects matching proposal shreds. A proposal shred matches
`ref` if:

```
shred.header.epoch          == ref.epoch
shred.header.proposal_cycle == ref.proposal_cycle
shred.header.proposer_index == ref.proposer_index
shred.header.pslice_index   == ref.pslice_index
shred.header.tx_list_hash   == ref.tx_list_hash
```

The leader decrypts forwarded proposal shreds using the attester keys from
`AttestationBundle`.

The leader needs at least `gamma_p` unique `shred_index` values matching the
same signed proposal header.

If fewer than `gamma_p` matching proposal shreds are available, the leader MUST
wait, choose a different valid attestation set, or skip producing this batch.

If at least `gamma_p` are available:

```
payload = reed_solomon_decode(erasure_root, pieces)
```

If decode fails, the leader MUST wait, choose a different valid attestation set,
or skip producing this batch.

The decoded payload MUST parse as:

```
ProposalSlicePayload { transactions }
```

The leader then checks:

- `tx_list_hash(transactions) == ref.tx_list_hash`
- all transactions are `ConstellationV1`
- all transactions pass structural and Constellation transaction checks from
  SIMD-XXXX
- `target_proposer_mask[ref.proposer_index] == 1` for every transaction
- `expire_cycle >= ref.proposal_cycle` for every transaction
- the proposal slice contains no duplicate transaction

If any check fails, the leader MUST wait, choose a different valid attestation
set, or skip producing this batch. Otherwise, the leader includes:

```
Submission { proposal_ref: ref, transactions }
```

### Batch validity

A `ConstellationBatch` is valid iff all of the following hold:

- `attester_indices` has exactly `alpha` bits set
- `attestations` are sorted by `attester_index`
- each attestation is valid
- each `AttestationPlus` signature verifies against the hashes of its full lists
- `submissions` are sorted in newly attested order
- for each submission, `submission.proposal_ref` is newly attested
- for each submission,
  `tx_list_hash(submission.transactions) == submission.proposal_ref.tx_list_hash`
- there are no extra submissions
- there are no missing submissions for newly attested proposal references
- `batch.cycle` is not a future cycle under the slot's replayed fork context

### Block-level cycle validity

Let `parent` be the parent Bank and block.

If the parent slot is the immediately previous slot:

```
first_batch.cycle == parent.highest_constellation_cycle + 1
```

If one or more slots were skipped:

```
first_batch.cycle > parent.highest_constellation_cycle
```

Inside the block:

```
batches[i].cycle == batches[i-1].cycle + 1
```

A block MAY contain zero batches only if normal Solana skip or empty-block rules
allow it.

### Block CU validity

For each batch, validators compute the deterministic transaction assignment
algorithm defined by SIMD-XXXX and obtain:

```
ms(batch) = max scheduled end time over all lanes, or 0 if no transactions scheduled
```

Block validity requires:

```
sum(ms(batch) for batch in block.batches) <= bcu
```

This prevents a leader from stuffing too many executable batches into one slot
after a catch-up period.

### Ledger inclusion

The leader serializes `ConstellationBlockPayload` into the slot's ledger
entries and broadcasts those entries as normal Solana ledger shreds. Proposal
shreds exchanged between proposers and attesters are not ledger shreds.

### Consensus voter validity

The voting protocol itself remains unchanged. The only change is the definition
of when a slot's Bank or fork is valid and therefore votable.

A validator MUST NOT vote for a slot unless all normal Solana checks and all
Constellation checks pass.

Normal checks include:

- ledger shreds are signed by the scheduled slot leader
- shreds reconstruct valid entries
- the fork chains to a valid parent
- the Bank can be replayed
- the resulting Bank hash is the fork hash being considered

Constellation-specific checks include:

- `ConstellationBlockPayload` parses
- `slot`, `parent_slot`, and `parent_bank_hash` match the fork context
- block cycle validity holds
- each batch has exactly `alpha` valid attestations
- all `AttestationPlus` records verify
- all newly attested proposal references have exactly one matching submission
- there are no extra submissions
- all submission `tx_list_hash` values match their proposal references
- all decoded proposal slices are valid
- the deterministic replay schedule is well formed
- sum of batch makespans is at most `bcu`
- no batch references a future cycle
- ReplayStage successfully executes the block payload and freezes the Bank

A failed check makes the slot or fork invalid for voting. The validator then
continues normal fork-choice, repair, and skip behavior.

Vote lockouts, fork choice, optimistic confirmation, and finalization rules are
otherwise unchanged.

