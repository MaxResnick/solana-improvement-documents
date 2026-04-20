---
simd: 'XXXX'
title: Epoch Role Schedules
authors:
  - (fill in with names of authors)
category: Standard
type: Core
status: Idea
created: 2026-04-20
feature: (fill in with feature key and github tracking issues once accepted)
---

## Summary

This proposal extends Solana's epoch schedule derivation to produce three
stake-weighted schedules at the start of each epoch in perperation for multiple concurrent proposers:

1. a leader schedule, keyed by slot, with one leader per slot
2. a proposer schedule, keyed by slot, with 16 ordered proposers per slot
3. an attester schedule, keyed by 50 ms cycle, with 256 ordered attesters per
   cycle

The proposer and attester committees are rolling committees. Each new slot replaces one proposer, and each new cycle replaces one attester.

## Motivation

The existing leader schedule gives the cluster a deterministic slot leader, but
it does not define proposer or attester assignments.

The goal of this proposal is to extend the leader schedule to include those assignments.

## Background

Today, Solana derives the leader schedule from a bank's epoch stake snapshot:

1. The snapshot contains the vote accounts, validator identities, and delegated
stake for the leader schedule epoch.
2. Validators filter out zero-stake vote accounts.
3. The epoch number is encoded into a 32-byte seed and used to initialize a ChaCha RNG.
4. Randomness from this RNG stream is used to select leaders by stake weight
5. The expanded leader schedule is cached in memory by epoch. It is not persisted
as a separate consensus object. On replay, restart, or snapshot restore,
validators recover the epoch stake snapshots from bank state and recompute the
same leader schedule.

## New Terminology

**Proposer committee** is the ordered set of 16 validators assigned to a slot.

**Attester committee** is the ordered set of 256 validators assigned to a
50 ms cycle.

**Cycle** is the 50 ms interval used to index attester assignments.

## Detailed Design


### Schedule derivation

At the epoch schedule boundary, validators derive the leader, proposer, and
attester schedules from the epoch stake distribution. The input stake
distribution maps each vote account to its validator identity and delegated
stake:

```
vote_pubkey -> (node_pubkey, stake)
```

Zero-stake entries are ignored. All three schedules use stake-weighted sampling with replacement over the
eligible validator set. The sampling randomness is generated from independent ChaCha RNG
streams. Each stream is seeded with:

```
SHA256("leader-schedule-v1"   || genesis_hash || epoch_le)
SHA256("proposer-schedule-v1" || genesis_hash || epoch_le)
SHA256("attester-schedule-v1" || genesis_hash || epoch_le)
```

### Leader schedule

The leader schedule remains slot-based. A lookup by slot returns the validator
identity responsible for leading that slot.

This proposal does not change the leader window duration it remains at 4.

### Proposer schedule

The proposer schedule is slot-based. A lookup by slot returns an ordered
committee of 16 proposers.

The first slot of the epoch is initialized by sampling with replacement from
the proposer RNG stream until 16 proposers have been selected. Duplicate
validator identities are allowed in the proposer committee.

The proposer committee advances by replacing one member
every 4 slots:

```
[p0, p1, ..., p15] -> [p1, p2, ..., p15, p16]
```

The new proposer is sampled from the proposer RNG stream and appended to the
end of the committee. It may already appear in the retained 15-member suffix.

### Attester schedule

The attester schedule is cycle-based. A lookup by cycle returns an ordered
committee of 256 attesters.

The first cycle of the epoch is initialized by sampling with replacement from
the attester RNG stream until 256 attesters have been selected. Duplicate
validator identities are allowed in the attester committee.

After the first cycle, the attester committee advances by replacing one member
per cycle:

```
[a0, a1, ..., a255] -> [a1, a2, ..., a255, a256]
```

The new attester is sampled from the attester RNG stream and appended to the
end of the committee. It may already appear in the retained 255-member suffix.

The attester schedule is therefore keyed by 50 ms cycle, not by slot, but its
contents are still derived entirely from the epoch stake snapshot.


### Storage

The expanded schedules are derived data. They do not need to be persisted.


## Alternatives Considered


## Impact

Validators will derive and cache two additional schedules alongside the leader
schedule.

## Security Considerations


## Drawbacks


## Backwards Compatibility

This proposal is backwards compatible as long as the leader window duration and
leader schedule semantics are unchanged.

On activation, the proposer and attester schedules may be introduced as unused
derived data. Validators can derive and cache the new role assignments without
using them for block production, block validation, fork choice, or rewards.
Later proposals can define how the proposer and attester roles are consumed.

## Open Questions
