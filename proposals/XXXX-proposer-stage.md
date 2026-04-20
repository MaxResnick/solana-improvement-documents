---
simd: 'XXXX'
title: Proposer Stage
authors:
  - Max Resnick (Anza)
category: Standard
type: Core
status: Review
created: 2026-04-20
feature:
---

## Summary

This proposal defines the proposer stage for multiple concurrent proposers.
Each proposer is a validator scheduled by `proposers_at(slot)` to contribute
transactions for that slot. A proposer receives targeted Multiple Concurrent
Proposer (MCP) transactions, performs bankless checks, assembles accepted
transactions into a proposer batch, erasure-encodes the batch into proposer
shreds, signs those shreds or a Merkle root that commits to them, and sends the
shreds to the attesters responsible for the relevant attestation cycle.

The proposer does not execute transactions, produce the replayable block, or
decide final transaction order. Later stages define attestation, leader
inclusion, reconstruction, ordering, execution, and fork choice.

## Motivation

Multiple concurrent proposers require a transaction ingress stage that can
filter obviously invalid or unpaid transactions before attesters and later
block-construction stages spend bandwidth and compute on them. The proposer
stage provides that first filter without giving proposers authority to execute
transactions, mutate account state, produce replayable blocks, or determine
final ordering.

Dedicated fee payer accounts give proposers a stronger pre-execution fee
guarantee. Because these accounts maintain a bonded minimum fee-payer balance
and accept additional withdrawal restrictions, proposers can initialize local
fee-payer caches without first observing prior transactions from the account.
MCP transactions that use a dedicated fee payer can therefore receive priority
inclusion over transactions whose fee-payer balances must be fetched and
discounted opportunistically.

The expected outcome is a deterministic wire and validation surface for
proposer batches. Clients can route MCP transactions to a targeted proposer,
receivers can authenticate proposer shreds, and later stages can reconstruct
candidate transaction batches from multiple proposers.

## Dependencies *(Optional)*

This proposal depends on the following proposal:

- **[SIMD-XXXX]: Epoch Role Schedules**

    Defines `proposers_at(...)`, the deterministic proposer committee lookup
    used to validate `target_proposer` and proposer-shred identity.

- **[SIMD-0385]: Transaction V1 Format**

    Defines the transaction format that MCP transactions extend.

[SIMD-XXXX]: ./XXXX-epoch-role-schedules.md
[SIMD-0385]: ./0385-transaction-v1.md

## New Terminology

**Proposer** is a validator scheduled by `proposers_at(slot)` to contribute
transactions for that slot.

**MCP transaction** is a Transaction V1-derived signed transaction that adds
required target cycle, target proposer, and last valid cycle fields.

**Target cycle** is the cycle for which an MCP transaction or proposer shred is
intended.

**Target proposer** is the proposer's index in the proposer committee for the
slot that contains `target_cycle`.

**Slot for cycle** is the deterministic mapping from a target cycle to the slot
whose proposer committee is responsible for that cycle.

**Proposer batch** is the ordered set of MCP transactions accepted by a
proposer for a target cycle.

**Proposer shred** is an erasure-coded fragment of a proposer batch. It is not
a leader block shred and cannot be inserted or replayed as a normal block
shred.

**Cycle-local balance cache** is the proposer-local fee-payer balance cache
used to reject transactions that cannot pay the base transaction fee and MCP
inclusion fee.

**Dedicated fee payer account** is a normal account that has opted into an
additional fee-payer bond. While the bond is active, the account is subject to
additional withdrawal, allocation, and ownership restrictions.

## Detailed Design

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in [RFC
2119](https://www.ietf.org/rfc/rfc2119.txt) and [RFC
8174](https://www.ietf.org/rfc/rfc8174.txt).

### Proposer role

A proposer:

1. receives MCP transactions for a signed target cycle
2. runs the bankless checks needed to reject obviously invalid or unpaid
   transactions
3. assembles accepted transactions into a proposer batch
4. erasure-encodes the batch into proposer shreds
5. signs the proposer shreds, or a Merkle root that commits to them
6. sends the proposer shreds to the attesters responsible for the relevant
   attestation cycle

The proposer MUST NOT execute transactions, produce the replayable block, or
decide final transaction order.

### MCP transaction format

An MCP transaction is a Transaction V1-derived signed transaction. It keeps the
Transaction V1 account, instruction, config, and signature layout, but replaces
Transaction V1's `LifetimeSpecifier` with required `target_cycle` and
`last_valid_cycle` config fields. These cycle fields define the transaction's
proposer-stage lifetime.

The encoded transaction is:

```
VersionByte
LegacyHeader
McpTransactionConfigMask
NumInstructions
NumAddresses
Addresses
ConfigValues
InstructionHeaders
InstructionPayloads
Signatures
```

`VersionByte` must identify the MCP transaction format. The exact byte is not
defined here, but it must be distinct from legacy, v0, and ordinary v1
transactions.

`McpTransactionConfigMask` is a `u32` bitmask with the same encoding rule as
Transaction V1: each set bit contributes one 4-byte little-endian value to
`ConfigValues`, in ascending bit order. Multi-word fields use adjacent bits,
and all bits for that field must be set.

The MCP transaction config fields are:

```
bits [0, 1]  inclusion_fee_lamports: u64
bits [2, 3]  order_fee_lamports: u64
bit  [4]     compute_unit_limit: u32
bit  [5]     loaded_accounts_data_size_limit: u32
bit  [6]     heap_size: u32
bits [7, 8]  target_cycle: u64
bit  [9]     target_proposer: u32
bits [10, 11] last_valid_cycle: u64
```

`inclusion_fee_lamports` is paid for including the transaction in a proposer
batch. If one of bits 0 or 1 is set, both must be set. If neither bit is set,
the inclusion fee is 0.

`order_fee_lamports` is used by later ordering logic to rank transactions
after batch reconstruction. If one of bits 2 or 3 is set, both must be set. If
neither bit is set, the order fee is 0.

The Transaction V1 resource fields are retained, but shifted to bits 4, 5, and
6 so that bits 0 through 3 can carry the two MCP fee values.

`target_cycle` is required. An MCP transaction is invalid if exactly one of
bits 7 or 8 is set, or if neither bit is set.

`target_proposer` is required. It is the proposer's index in the proposer
committee for the slot that contains `target_cycle`, encoded as a little-endian
`u32`. The transaction is invalid if bit 9 is not set, or if `target_proposer`
is not a valid index for the target cycle's proposer committee.

`last_valid_cycle` is required. It is the last cycle in which the target
proposer may accept the transaction, encoded as a little-endian `u64`. The
transaction is invalid if exactly one of bits 10 or 11 is set, or if neither
bit is set. The transaction is invalid if `last_valid_cycle < target_cycle`.

The signed message is every field before `Signatures`, including
`McpTransactionConfigMask`, `ConfigValues`, and therefore `target_cycle`,
`target_proposer`, and `last_valid_cycle`. `Signatures[i]` signs that message
with the key for `Addresses[i]`, as in Transaction V1.

All other Transaction V1 constraints apply unless changed here:

- no address lookup tables
- no duplicate addresses
- at most 64 account addresses
- at most 64 instructions
- at most 12 signatures
- at most 4096 encoded bytes

ComputeBudgetProgram instructions do not configure MCP transactions. Fee and
resource requests come from `McpTransactionConfigMask` and `ConfigValues`.

### Dedicated fee payer accounts

MCP transactions MAY mark their fee payer as a dedicated fee payer account.
Transaction V1 account indexes use `u8` values, while MCP transactions retain
the Transaction V1 limit of at most 64 account addresses. Therefore, only the
low six bits are needed to identify an address. MCP transactions reuse one of
the unused high bits as a dedicated-fee-payer marker:

```
const ACCOUNT_INDEX_MASK: u8 = 0x3f
const DEDICATED_FEE_PAYER_ACCOUNT: u8 = 0x40
```

When decoding an account index, the address index is `account_index &
ACCOUNT_INDEX_MASK`. If `DEDICATED_FEE_PAYER_ACCOUNT` is set, the decoded
address index MUST be the transaction fee payer index. The transaction is
invalid if this bit is set on any other account index. The remaining high bit
is reserved and MUST be unset.

If no account index referring to the fee payer carries
`DEDICATED_FEE_PAYER_ACCOUNT`, the proposer treats the transaction as using a
standard fee payer.

A dedicated fee payer account is otherwise a normal account, but it carries an
additional fee-payer bond. The account's bonded minimum balance is the balance
that proposers MAY use to initialize their cycle-local fee-payer balance cache
before seeing any prior transactions for that account in the target cycle.

While the dedicated fee-payer bond is active:

- The account MUST NOT allocate data.
- The account MUST NOT transfer ownership.
- The account MUST NOT withdraw below its bonded minimum balance except through
  the dedicated fee-payer withdrawal path.
- A dedicated fee-payer withdrawal MUST be held for at least 400 ms before the
  withdrawn lamports become available.

The withdrawal hold gives in-flight proposers time to stop relying on the old
bonded minimum balance. During the hold, proposers continue to treat the
pre-withdrawal bonded minimum as the account's dedicated fee-payer cache floor.

### Fee payer check logic

The proposer performs fee-payer checks against a candidate parent bank for the
target cycle. This bank is the proposer's base bank for the target cycle.

For each received MCP transaction, the proposer:

1. verifies that the signed `target_cycle` matches the cycle being proposed
2. verifies that the signed `target_proposer` matches the proposer's scheduled
   index in `proposers_at(slot_for_cycle(target_cycle))`
3. verifies that the current cycle is less than or equal to
   `last_valid_cycle`
4. sanitizes the transaction
5. verifies all required signatures
6. computes the fee from the base bank fee rules, including the MCP inclusion
   fee
7. checks the fee payer balance using the cycle-local balance cache
8. accepts the transaction into the proposer batch only if the cache can pay
   the computed fee

The fee payer is the first writable signer, matching Transaction V1 account
ordering. The proposer fee-payer check charges the base transaction fee and
`inclusion_fee_lamports`. The `order_fee_lamports` value is signed and carried
for later ordering logic, but it is not charged by the proposer-stage
fee-payer check unless a later proposal explicitly requires it.

The balance cache is scoped to:

```
target_cycle
base_bank_hash
proposer_index
```

On first use of a fee payer, the proposer loads the fee payer account from the
base bank. If the transaction marks the fee payer as a dedicated fee payer
account, the proposer verifies that the account is a valid dedicated fee payer
account and initializes the cache with the account's bonded minimum balance.
Otherwise, the proposer stores the account's lamport balance in the cache. If
the cached balance is less than the computed fee, the proposer drops the
transaction. Otherwise, the proposer subtracts the fee from the cached balance
and may add the transaction to the batch.

If any account in the transaction may be debited during execution, and that
account is used as an instruction account, the proposer conservatively sets
that account's cached balance to zero after accepting the transaction. This
prevents the proposer from repeatedly accepting transactions that rely on a
balance that execution may spend before replay reaches later transactions.

The proposer must not execute the transaction and must not apply account writes
to the base bank. Passing the proposer fee-payer check only means the
transaction is eligible for proposer-stage batching. Replay still performs the
normal transaction checks and execution.

### Proposer shred format

A proposer shred is not a leader block shred. It must use a distinct wire
variant and a distinct identity namespace, so it cannot be inserted or replayed
as a normal block shred.

Each proposer shred contains:

```
ProposerShred {
    variant: u8,
    target_cycle: u64,
    base_bank_hash: [u8; 32],
    proposer_identity: [u8; 32],
    proposer_index: u8,
    batch_id: u32,
    fec_set_index: u32,
    shred_index: u32,
    shred_kind: u8,
    num_data_shreds: u16,
    num_coding_shreds: u16,
    payload_size: u16,
    payload: [u8; payload_size],
    signature: [u8; 64],
}
```

`variant` identifies the proposer-shred wire format.

`target_cycle` is the cycle the batch is intended for.

`base_bank_hash` identifies the base bank used for proposer fee-payer checks.

`proposer_identity` is the validator identity that signed the shred.

`proposer_index` is the proposer's position in the proposer committee for the
slot that contains `target_cycle`. If a validator appears more than once in the
proposer committee, each occurrence is a separate proposer index.

`batch_id` is chosen by the proposer and is monotonically increasing per
`(target_cycle, proposer_index)`.

`fec_set_index`, `shred_index`, `shred_kind`, `num_data_shreds`, and
`num_coding_shreds` identify the erasure set and the shred's position within
that set.

`payload` is a fragment of the encoded proposer batch. The batch contains MCP
transactions in proposer-chosen order. The payload must not contain executed
account state.

`signature` is produced by `proposer_identity` over every prior field in the
encoded proposer shred. A future Merkle-root variant may replace per-shred
signatures, but the signed root must commit to all fields above.

A receiver accepts a proposer shred only if:

- `target_cycle` is in its acceptable window
- `proposer_identity` appears at `proposer_index` in
  `proposers_at(slot_for_cycle(target_cycle))`
- `signature` is valid
- `shred_kind` is a known proposer data or proposer coding kind
- the shred identity is not an inconsistent duplicate

The proposer shred identity is:

```
target_cycle
proposer_index
batch_id
fec_set_index
shred_index
shred_kind
```

Two proposer shreds with the same identity but different signed contents are
inconsistent duplicates.

## Alternatives Considered

Transactions could be gossiped to proposers without a signed target proposer.
That design gives clients less routing control and makes it harder for
receivers to determine whether a proposer was authorized to accept a
transaction for a specific slot.

The protocol could also rely on leader-only prefiltering. That would avoid a
new proposer transaction and shred path, but it would not support parallel
transaction contribution by multiple proposers before later reconstruction and
ordering stages.

Proposers could build batches without fee-payer checks. That would simplify the
proposer role, but it would let unpaid transactions consume proposer,
attester, and reconstruction bandwidth before replay rejects them.

Dedicated fee payer accounts could be represented with a separate transaction
config field instead of an account-index marker. Reusing an unused account-index
bit avoids spending another config bit and keeps the marker attached to the
account it qualifies, at the cost of making account-index decoding MCP-specific.

## Impact

### Validators

Validators that act as proposers need to accept targeted MCP transactions,
maintain cycle-local balance caches, build proposer batches, erasure-encode
those batches into proposer shreds, and sign the resulting shreds or Merkle
roots. They also need to recognize dedicated fee payer accounts and initialize
the local fee-payer cache from the bonded minimum balance when the transaction
uses the dedicated fee-payer marker.

Validators that receive proposer shreds need to validate proposer identity,
target cycle, proposer index, signature, shred kind, and duplicate identity
before forwarding the data to later attestation or reconstruction stages.

### Dapp developers and clients

Clients submitting MCP transactions need to select a target cycle and target
proposer, sign those fields, and include MCP fee and resource fields in the MCP
transaction config mask instead of ComputeBudgetProgram instructions. Clients
that want priority inclusion through a dedicated fee payer need to opt the fee
payer account into the dedicated fee-payer bond and set the dedicated fee-payer
account-index marker.

### Core contributors

Core protocol implementations need new transaction parsing, proposer-shred wire
types, proposer schedule lookups, fee-payer balance-cache logic, and duplicate
proposer-shred detection. Runtime implementations also need dedicated
fee-payer account state and enforcement for withdrawal, allocation, and
ownership restrictions.

## Security Considerations

The target cycle, target proposer, and last valid cycle are signed transaction
fields. A proposer or relay cannot retarget a transaction to another proposer
or cycle without invalidating the transaction signatures.

The proposer fee-payer check is only a prefilter. Replay MUST still perform
normal transaction checks and execution, because proposers do not apply account
writes to the base bank and may only have a conservative balance-cache view.

Dedicated fee payer accounts make proposer fee-payer checks stronger, but only
if the bonded minimum balance cannot be withdrawn or invalidated faster than
proposers can stop relying on it. The 400 ms withdrawal hold, allocation
restriction, and ownership restriction are part of that guarantee.

Proposer shreds use a distinct wire variant and identity namespace. This
prevents proposer shreds from being inserted or replayed as normal leader block
shreds.

Receivers MUST reject inconsistent duplicate proposer shreds with the same
identity and different signed contents. These duplicates are evidence that the
same proposer index equivocated for the same target cycle, batch, FEC set,
shred index, and shred kind.

## Drawbacks *(Optional)*

This adds a new transaction format, proposer-shred format, and fee-payer
prefilter path. It also requires clients to target specific proposers and
cycles, which creates new routing and retry complexity.

Dedicated fee payer accounts also add a new account mode and impose liquidity
costs on users who want priority inclusion. The 400 ms withdrawal hold improves
fee certainty for proposers, but delays access to bonded lamports.

## Backwards Compatibility *(Optional)*

This proposal introduces a distinct MCP transaction version and a distinct
proposer-shred wire variant. Existing legacy, v0, and ordinary v1 transactions
are unchanged. Existing block shreds are unchanged because proposer shreds are
not replayable block shreds.
