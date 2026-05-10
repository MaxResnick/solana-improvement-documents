---
simd: 'XXXX'
title: Constellation Role Schedules and Common Parameters
authors:
  - (fill in with names of authors)
category: Standard
type: Core
status: Idea
created: 2026-04-20
feature: (fill in with feature key and github tracking issues once accepted)
---

## Summary

This proposal defines the common notation, constants, wall-clock cycle index,
proposal-slice index, hashes, and epoch-derived role schedules used by
Constellation.

Constellation uses the normal Solana slot leader as the consensus leader and
adds two cycle-indexed schedules:

```
proposer(e, c)[j] -> Option<ValidatorPubkey>, 0 <= j < p
attester(e, c)[k] -> ValidatorPubkey,         0 <= k < q
```

These schedules are derived from the same finalized Bank state that is used to
compute the Solana leader schedule for the epoch.

## New Terminology

A `cycle` is a wall-clock interval of `Delta_c` nanoseconds. The default cycle
duration is 50 ms.

A `consensus leader` is the normal Solana slot leader for a slot.

A `proposer` creates proposal slices for a cycle and sends proposal shreds to
attesters.

An `attester` receives proposal shreds for a cycle, forwards them to upcoming
consensus leaders, and signs attestation lists.

A `replay validator` replays a block payload and computes the resulting Bank
state.

A `proposal slice` is an ordered transaction list created by a proposer. A
proposer may create at most `mu` proposal slices per cycle.

A `proposal shred` is a proposer-to-attester erasure-coded message, distinct
from a Solana ledger shred.

## Detailed Design

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in [RFC
2119](https://www.ietf.org/rfc/rfc2119.txt) and [RFC
8174](https://www.ietf.org/rfc/rfc8174.txt).

### Parameters

Constellation uses the following default paper parameters:

| Parameter | Default | Meaning |
| - | - | - |
| `p` | 16 | Proposers per cycle |
| `q` | 256 | Attesters per cycle |
| `Gamma_p` | `q` | Proposal shreds created per proposal slice |
| `gamma_p` | `q / 4` | Proposal shreds required to reconstruct a proposal slice |
| `alpha` | `q / 2` | Attestations required in a batch |
| `mu` | 4 | Maximum proposal slices per proposer per cycle |
| `Delta` | protocol value | Message delay bound used to size attestation windows |
| `Delta_c` | 50 ms | Cycle duration |
| `lambda` | `ceil(Delta / Delta_c) + 1`, default 6 | Attestation-window length |
| `tau` | 32 cycles | Role replacement interval |
| `kappa` | 3 | Number of future leaders that attesters forward shreds to |
| `tcu` | 1.4M CUs | Maximum transaction CU limit |
| `mcu` | 2M CUs | Batch makespan limit per execution lane |
| `bcu` | 32M CUs | Block sum-of-batch-makespans limit |
| `m` | 4 | Deterministic execution lanes |
| `phi` | 0.001 SOL | Fee-payer reserve floor |
| `upsilon` | protocol value | Common-case finalized-observation bound, in cycles |
| `U` | `upsilon + lambda` | Inclusion-fee reserve window |

`Delta` is a protocol parameter, not a locally measured value. Validators use
the activated value of `Delta` when deriving `lambda`.

### Cycle index

Cycles are wall-clock driven:

```
cycle(now) = floor(unix_timestamp_nanoseconds / 50_000_000)
```

Slots remain the Solana consensus units. A block MAY contain multiple
Constellation batches, where each batch corresponds to one cycle.

### Proposal slice index

Proposal slice indices use Solana-style zero-based indexing:

```
pslice_index t = proposal_cycle * mu + local_slice_index
0 <= local_slice_index < mu
slice_cycle(t) = floor(t / mu)
```

This is the zero-based equivalent of the paper's proposal-slice formula.

### Hash functions

Constellation uses the following transaction-list commitment notation:

```
tx_hash(tx)     = hash(serialized_sanitized_transaction_bytes)
tx_list_hash(T) = hash(tx_hash(tx_0) || tx_hash(tx_1) || ... || tx_hash(tx_n))
```

`tx_list_hash` is the commitment that constrains the consensus leader when the
leader reconstructs a proposal slice from proposal shreds.

### Epoch source of truth

For epoch `e`, the validator set and stake distribution are taken from the
finalized Bank state used to compute the Solana leader schedule for that epoch.

Constellation adds two schedules to the normal slot leader schedule:

```
leader(e, slot)      -> ValidatorPubkey
proposer(e, c)[j]    -> Option<ValidatorPubkey>, 0 <= j < p
attester(e, c)[k]    -> ValidatorPubkey,         0 <= k < q
```

The expanded schedules are derived data. Validators recover the epoch stake
snapshots from Bank state and recompute the same role schedules on replay,
restart, or snapshot restore.

### Epoch seed

The Constellation epoch seed is:

```
epoch_seed = hash("constellation-epoch" || genesis_hash || epoch)
```

The first active proposer vector and the first active attester vector for an
epoch are derived by sampling from this seed with the index-specific domains
defined below. Replacement rounds then update one index at a time.

### Consensus leader schedule

The consensus leader is the normal Solana slot leader for the slot. The slot
leader signs the produced ledger shreds, and validators verify the leader
against their local leader schedule.

### Proposer schedule

For each epoch `e`, validators maintain a vector of `p` proposer slots.

Every `tau` cycles, exactly one proposer index is rotated:

```
replacement_round(c) = floor(c / tau)
replaced_index(c)    = replacement_round(c) mod p
```

At replacement boundary `r`, index `j = replaced_index(r)` changes as follows:

```
old proposer active through cycle r - 1
proposer(e, c)[j] = None for r <= c < r + lambda
new proposer active from cycle r + lambda onward
```

The `lambda`-cycle gap is REQUIRED. It ensures that an attestation list for
proposer index `j` over the last `lambda` cycles cannot contain proposal
commitments from two different proposer identities.

The new proposer is sampled stake-weighted from the epoch validator set using
the deterministic seed:

```
seed = hash(
    "constellation-proposer" ||
    epoch ||
    proposer_index ||
    replacement_round ||
    epoch_seed
)
```

Sampling MUST be without replacement within the active proposer set. If the
sampled validator is already active as another proposer index in the same
cycle, validators resample by incrementing a counter in the seed domain.

### Attester schedule

For each epoch `e`, validators maintain a vector of `q` attester slots.

Every `tau` cycles, exactly one attester index is rotated:

```
replacement_round(c) = floor(c / tau)
replaced_index(c)    = replacement_round(c) mod q
```

Attesters do not require the proposer-style `lambda`-cycle gap. The expected
attester for an attestation is determined solely by:

```
(epoch, attestation_cycle, attester_index)
```

The new attester is sampled stake-weighted from the epoch validator set using
the deterministic seed:

```
seed = hash(
    "constellation-attester" ||
    epoch ||
    attester_index ||
    replacement_round ||
    epoch_seed
)
```

Sampling MUST be without replacement within the active attester set for a
cycle. If the sampled validator is already active as another attester index in
the same cycle, validators resample by incrementing a counter in the seed
domain.

### Epoch boundary behavior

Near an epoch transition, nodes may not agree yet on whether wall-clock cycle
`c` belongs to epoch `e` or `e + 1`. During this ambiguity window, proposers and
attesters for both epochs MAY be active.

Proposal shreds and attestations MUST include the epoch. Consensus and replay
validity are determined by the parent Bank's epoch schedule and the slot's fork
context. A block payload is valid only if its batches reference the epoch that
is valid for the Bank and fork being replayed.


