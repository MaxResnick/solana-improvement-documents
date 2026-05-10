---
simd: 'XXXX'
title: Constellation Attester Stage
authors:
  - (fill in with names of authors)
category: Standard
type: Core
status: Idea
created: 2026-05-11
feature: (fill in with feature key and github tracking issues once accepted)
extends: SIMD-XXXX, SIMD-XXXX
---

## Summary

This proposal defines the Constellation attester stage. Attesters receive
proposal shreds from scheduled proposers, validate the shred identity and
commitment, forward encrypted shreds to upcoming consensus leaders, and sign
per-proposer attestation lists at cycle boundaries.

## New Terminology

An `attestation cycle` is the cycle for which an attester signs its attestation
lists.

A `proposal commitment` is the tuple committed to by an attester for one
proposal slice:

```
(proposal_cycle, pslice_index, tx_list_hash)
```

A `attestation list` is a sorted list of proposal commitments for one proposer
index.

A `attestation bundle` is the attestation, the full lists that hash to the
attestation, and the symmetric key needed by leaders to decrypt forwarded
proposal shreds for the attester and cycle.

## Detailed Design

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in [RFC
2119](https://www.ietf.org/rfc/rfc2119.txt) and [RFC
8174](https://www.ietf.org/rfc/rfc8174.txt).

This proposal depends on the schedules and common parameters defined by
SIMD-XXXX and the proposal shred format defined by SIMD-XXXX.

### Attester receive checks

At attester cycle `c_att`, attester index `k` accepts a `ProposalShred ps` for
attestation only if all of the following checks pass:

- `ps` parses
- `ps.header.epoch` is active for the attester's fork context
- `ps.header.proposer_index < p`
- `ps.shred_index == k`
- `ps.header.pslice_index / mu == ps.header.proposal_cycle`
- `proposer(ps.header.epoch, ps.header.proposal_cycle)[ps.header.proposer_index]`
  is not `None`
- the proposer schedule maps that index to the pubkey used to verify
  `ps.proposer_signature`
- `proposer_signature` verifies over `ProposalSliceHeader` with that pubkey
- `merkle_path` verifies `erasure_piece` under `erasure_root`
- the proposal cycle is inside the attestation window:

```
c_att - lambda + 1 <= ps.header.proposal_cycle <= c_att
```

If the activated protocol includes a clock-skew tolerance `Delta_skew`, every
validator uses the same widened window:

```
lambda_skew_low  = ceil((Delta + Delta_skew) / Delta_c)
lambda_skew_high = ceil(Delta_skew / Delta_c)

c_att - lambda_skew_low + 1 <= proposal_cycle <= c_att + lambda_skew_high
```

### Duplicate and equivocating proposal shreds

For attestation purposes, an honest attester signs at most one commitment per:

```
(epoch, proposer_index, pslice_index)
```

If multiple validly signed proposal shreds arrive for the same tuple but with
different signed headers, the attester:

- forwards all such shreds to upcoming leaders
- attests only the first valid commitment it accepted for that tuple

### Forwarded encrypted proposal shred

For every proposal shred accepted for attestation, the attester immediately
forwards it to the next `kappa` consensus leaders for the attester's fork
context. If the attester observes conflicting valid shreds for the same
`(epoch, proposer_index, pslice_index)`, it forwards each conflicting shred.

For each attestation cycle `c_att`, attester `k` creates a fresh symmetric key:

```
sk[e, c_att, k]
```

The attester sends:

```
struct EncryptedProposalShred {
    epoch: Epoch,
    attestation_cycle: u64,
    attester_index: u16,
    nonce: [u8; 12],
    ciphertext: Vec<u8>, // AEAD(ProposalShred)
}
```

The AEAD associated data is:

```
domain("constellation-forwarded-pshred") ||
epoch ||
attestation_cycle ||
attester_index
```

### Attestation message format

For attestation cycle `c`, attester index `k` forms `p` lists:

```
struct ProposalCommitment {
    proposal_cycle: u64,
    pslice_index: u64,
    tx_list_hash: Hash,
}

type AttestationList = Vec<ProposalCommitment>;

struct Attestation {
    epoch: Epoch,
    attestation_cycle: u64,
    attester_index: u16,
    list_hashes: [Hash; p],
    signature: Signature,
}
```

Each `L[j]` is the sorted list of commitments accepted from proposer index `j`
whose proposal cycle lies in the attestation window.

The sorting order is:

```
(proposal_cycle, pslice_index, tx_list_hash)
```

No duplicates are allowed for the same `(proposal_cycle, pslice_index)` in a
single `L[j]`.

The attester signs:

```
signature = Sign(
    attester_key,
    domain("constellation-attestation") ||
    epoch ||
    attestation_cycle ||
    attester_index ||
    hash(L[0]) ||
    ... ||
    hash(L[p-1])
)
```

The full attestation bundle sent to leaders is:

```
struct AttestationBundle {
    attestation: Attestation,
    lists: [AttestationList; p],
    symmetric_key: SymmetricKey, // sk[e,c,k]
}
```

The symmetric key allows the leader to decrypt previously forwarded proposal
shreds for that attester and cycle.

### Attestation validity

An attestation is valid only if all of the following hold:

- `attester_index < q`
- `attester(epoch, attestation_cycle)[attester_index]` is the signer
- the signature verifies
- all included lists hash to `list_hashes`
- each list is sorted
- each list contains no duplicate `pslice_index` for the same proposal cycle
- each commitment's `pslice_index / mu == proposal_cycle`
- each commitment lies inside the valid attestation window
- each referenced proposer index maps to a non-`None` proposer for the proposal
  cycle

