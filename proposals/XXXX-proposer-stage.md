---
simd: 'XXXX'
title: Constellation Proposer Stage
authors:
  - (fill in with names of authors)
category: Standard
type: Core
status: Idea
created: 2026-05-11
feature: (fill in with feature key and github tracking issues once accepted)
extends: SIMD-XXXX
---

## Summary

This proposal defines the Constellation proposer stage. A proposer receives
Constellation transactions, performs the bankless checks needed for proposal
eligibility, assembles ordered proposal slices, erasure-codes those slices into
proposal shreds, signs the proposal-slice header, and sends one shred to each
scheduled attester.

## New Terminology

A `Constellation transaction` is a signed Solana transaction whose message uses
`VersionedMessage::ConstellationV1`.

A `proposal slice payload` is the ordered transaction list committed to by a
proposer.

A `proposal slice header` is the signed metadata for a proposal slice, including
the proposer index, proposal cycle, proposal slice index, transaction-list hash,
and erasure root.

A `proposal shred` is one erasure-coded piece of a proposal slice sent from the
proposer to the attester at the same index as the shred.

## Detailed Design

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in [RFC
2119](https://www.ietf.org/rfc/rfc2119.txt) and [RFC
8174](https://www.ietf.org/rfc/rfc8174.txt).

This proposal depends on the role schedules, cycle notation, proposal-slice
indexing, and common parameters defined by SIMD-XXXX.

### Transaction format

Constellation introduces a new versioned transaction message:

```
VersionedMessage::ConstellationV1
```

The existing Solana transaction envelope remains unchanged:

```
struct VersionedTransaction {
    signatures: Vec<Signature>,
    message: VersionedMessage,
}
```

A `ConstellationV1` message extends the current versioned Solana message with
Constellation metadata:

```
struct ConstellationMessageV1 {
    // Existing Solana message fields.
    header: MessageHeader,
    account_keys: Vec<Pubkey>,
    recent_blockhash: Hash,
    instructions: Vec<CompiledInstruction>,
    address_table_lookups: Vec<MessageAddressTableLookup>,

    // New Constellation fields, signed as part of the message.
    constellation: ConstellationTxMeta,
}

struct ConstellationTxMeta {
    target_proposer_mask: BitVec<p>,
    expire_cycle: u64,
    inclusion_bid_lamports_per_byte: u64,
}
```

The priority bid and compute-unit limit use existing Solana compute-budget
instructions:

```
priority_bid = ComputeBudget::SetComputeUnitPrice
cu_limit     = ComputeBudget::SetComputeUnitLimit, clamped to tcu
```

This keeps Constellation priority fees on Solana's current model: priority fee
is derived from requested CU limit and CU price.

### Transaction validity additions

A `ConstellationV1` transaction is valid only if all of the following hold:

- `target_proposer_mask` has length `p`
- at least one target proposer bit is set
- the effective CU limit is at most `tcu`
- the fee payer is `account_keys[0]`, writable, signer, and valid under normal
  Solana fee-payer rules
- the `recent_blockhash` or durable nonce is valid under normal Solana rules

The fee payer remains the first account and first signature, matching existing
Solana transaction semantics.

### Proposal slice payload

A proposer builds proposal slices. A proposal slice contains an ordered list of
Constellation transactions:

```
struct ProposalSlicePayload {
    transactions: Vec<VersionedTransaction>, // all must be ConstellationV1
}
```

For a proposal slice payload, the proposer computes:

```
h  = tx_list_hash(transactions)
rt = Merkle root of the Reed-Solomon encoded payload pieces
```

### Proposal slice header

The proposal slice header is:

```
struct ProposalSliceHeader {
    epoch: Epoch,
    proposal_cycle: u64,
    proposer_index: u16,
    pslice_index: u64,
    tx_list_hash: Hash,
    erasure_root: Hash,
}
```

The proposer signs:

```
sig = Sign(
    proposer_key,
    domain("constellation-pslice") || ProposalSliceHeader
)
```

### Proposal shred format

A proposal shred is a proposer-to-attester message:

```
struct ProposalShred {
    header: ProposalSliceHeader,
    shred_index: u16,          // 0 <= shred_index < q
    erasure_piece: Vec<u8>,
    merkle_path: MerklePath,
    proposer_signature: Signature,
}
```

Proposal shreds have the following validity constraints:

- `header.pslice_index / mu == header.proposal_cycle`
- `header.proposer_index < p`
- `shred_index < q`

Each proposer creates `q` proposal shreds and sends:

```
ProposalShred[shred_index = k] -> attester(epoch, proposal_cycle)[k]
```

### Proposer-local checks

A proposer uses a Bank derived from the latest finalized fork it has observed.
For proposal cycle `c`, the proposer MUST NOT propose if it has not observed a
finalized Bank whose highest Constellation cycle is at least:

```
c - upsilon
```

For each candidate transaction `tx`, proposer `j` performs the checks below
before inclusion. These checks are proposer-local filters. They reduce invalid
and unpaid proposals but do not replace the consensus-verifiable checks applied
by the leader and replay validators.

Structural checks:

- deserialize `VersionedTransaction`
- verify `tx.message` is `ConstellationV1`
- sanitize the transaction
- verify signatures
- resolve address lookup tables against the proposer base Bank
- parse compute-budget instructions
- reject duplicate or invalid compute-budget instructions
- reject if effective `cu_limit > tcu`
- reject if serialized transaction size exceeds Solana's packet-size limit
- reject if the transaction is already processed in the proposer base Bank's
  status cache

Constellation checks:

- reject if `target_proposer_mask[j] == 0`
- reject if `expire_cycle < proposal_cycle`
- reject if `inclusion_bid_lamports_per_byte` overflows fee arithmetic
- reject if `recent_blockhash` is not valid in the proposer base Bank, unless a
  durable nonce is valid

### Proposer-local fee-payer reserve checks

Let:

```
fp               = tx.account_keys[0]
bytes            = serialized_len(tx)
inclusion_fee_j  = bytes * tx.inclusion_bid_lamports_per_byte
U                = upsilon + lambda
reserve(fp)      = current reserve balance for fp in proposer base Bank
window_debt(fp,j,c) = sum of inclusion_fee_j for all transactions
                      proposer j included for fp in proposal cycles [c - U + 1, c]
```

The proposer MUST reject `tx` if any of the following hold:

- `fp` is missing
- `fp` fails Solana fee-payer validation
- `reserve(fp) < phi`
- `fp` has a pending reserve close or decrease effective at or before `c + U`
- `window_debt(fp,j,c) + inclusion_fee_j > reserve(fp) / p`

The proposer MAY also reject transactions that, in the proposer's observed Bank,
fail expected base-fee and priority-fee payment checks. Replay remains
authoritative for fee debits and execution.

### Proposal slice validity

A proposal slice is valid only if all of the following hold:

- every contained transaction passes the structural and Constellation checks in
  this proposal
- no transaction appears twice in the same proposal slice
- `tx_list_hash` equals `tx_list_hash(transactions)`
- the payload decodes under `erasure_root`

The reserve-window checks are proposer-local admission rules. Consensus
validation uses the Bank state and proposal history available on the replayed
fork, as specified by the leader and replay-stage SIMDs.

### Proposer send procedure

For each cycle `c` in which validator `v` is `proposer(e, c)[j]`:

```
for local_slice_index in 0..mu-1:
    txs = select locally accepted transactions
    if txs is empty: continue

    t  = c * mu + local_slice_index
    h  = tx_list_hash(txs)
    rt, pieces = reed_solomon_encode_with_merkle(
        txs,
        Gamma_p=q,
        gamma_p=q/4
    )

    sign ProposalSliceHeader(e, c, j, t, h, rt)

    for k in 0..q-1:
        send ProposalShred(..., shred_index=k, piece=pieces[k])
            to attester(e,c)[k]

    wait until next slice subinterval inside cycle
```

The proposer chooses which valid transactions to include.

