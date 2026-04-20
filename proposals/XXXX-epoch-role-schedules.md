---
simd: 'XXXX'
title: Epoch Role Schedules
authors:
  - Max Resnick (Anza)
category: Standard
type: Core
status: Review
created: 2026-04-20
feature:
---

## Summary

This proposal extends Solana's epoch schedule derivation to produce three
stake-weighted schedules at the start of each epoch in preparation for multiple
concurrent proposers:

1. a leader schedule, keyed by slot, with one leader per slot
2. a proposer schedule, keyed by slot, with 16 ordered proposers per slot
3. an attester schedule, keyed by 50 ms cycle, with 256 ordered attesters per
   cycle

The proposer and attester committees are rolling committees. Each new leader
window replaces one proposer, and each new cycle replaces one attester.

## Motivation

The existing leader schedule gives the cluster a deterministic slot leader, but
it does not define proposer or attester assignments.

The goal of this proposal is to extend the leader schedule to include those
assignments. Any protocol that uses multiple concurrent proposers or a sampled
attester committee needs all validators to agree on the same role assignments
during block production, replay, restart, and snapshot restore.

## Background

Today, Solana derives the leader schedule from a bank's epoch stake snapshot:

1. The snapshot contains the vote accounts, validator identities, and delegated
   stake for the leader schedule epoch.
2. Validators filter out zero-stake vote accounts.
3. The epoch number is encoded into a 32-byte seed and used to initialize a
   ChaCha RNG.
4. Randomness from this RNG stream is used to select leaders by stake weight.
5. The expanded leader schedule is cached in memory by epoch. It is not
   persisted as a separate consensus object. On replay, restart, or snapshot
   restore, validators recover the epoch stake snapshots from bank state and
   recompute the same leader schedule.

## New Terminology

**Proposer committee** is the ordered set of 16 validators assigned to a slot.

**Attester committee** is the ordered set of 256 validators assigned to a
50 ms cycle.

**Cycle** is the 50 ms interval used to index attester assignments.

## Detailed Design

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in [RFC
2119](https://www.ietf.org/rfc/rfc2119.txt) and [RFC
8174](https://www.ietf.org/rfc/rfc8174.txt).

### Schedule derivation

At the epoch schedule boundary, validators derive the leader, proposer, and
attester schedules from the epoch stake distribution. The input stake
distribution maps each vote account to its validator identity and delegated
stake:

```
vote_pubkey -> (node_pubkey, stake)
```

Zero-stake entries are ignored. All three schedules use stake-weighted sampling
with replacement over the eligible validator set. The sampling randomness is
generated from independent ChaCha RNG streams. Each stream is seeded with:

```
SHA256("leader-schedule-v1"   || genesis_hash || epoch_le)
SHA256("proposer-schedule-v1" || genesis_hash || epoch_le)
SHA256("attester-schedule-v1" || genesis_hash || epoch_le)
```

### Leader schedule

The leader schedule remains slot-based. A lookup by slot returns the validator
identity responsible for leading that slot.

This proposal does not change the leader window duration; it remains at four
slots.

### Proposer schedule

The proposer schedule is slot-based. A lookup by slot returns an ordered
committee of 16 proposers.

The first slot of the epoch is initialized by sampling with replacement from
the proposer RNG stream until 16 proposers have been selected. Duplicate
validator identities are allowed in the proposer committee.

The proposer committee advances by replacing one member every four slots:

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

The expanded schedules are derived data. They do not need to be persisted as a
new consensus object.

Validators persist the epoch stake snapshot as part of bank and snapshot state,
then recompute and cache the schedules by epoch. The expected lookup surface is:

```
leader_at(slot)
proposers_at(slot)
attesters_at_cycle(cycle)
```

## Alternatives Considered

The proposer and attester schedules could be sampled independently for every
slot or cycle. That design is simpler to describe, but creates complete
committee turnover at every boundary. Rolling committees retain most committee
members across adjacent slots or cycles while still rotating membership over
time.

The proposer and attester schedules could also be derived locally by each
subsystem that needs them. This proposal instead defines common schedule
derivation from the epoch stake snapshot so replay, restart, and snapshot
restore produce the same assignments across validators.

## Impact

Validators will derive and cache two additional schedules alongside the leader
schedule. Components that need proposer or attester assignments can use the
slot and cycle lookup APIs instead of constructing their own local assignment
logic.

The proposal does not require a new on-chain account format. It changes how
validators consume the epoch stake snapshot.

## Security Considerations

The main security requirement is deterministic derivation. Candidate ordering,
RNG seeding, eligible-set construction, and cycle selection must not depend on
local process state or map iteration order.

The eligible-set rules are part of consensus. Leaders, proposers, and attesters
are sampled with replacement and may appear more than once across their
schedules.

## Drawbacks

This adds more derived schedule state to validator memory and more schedule
logic for client implementations to match.

The rolling committee rule is more complex than independently sampling each
slot or cycle, but it gives smoother membership changes.

## Backwards Compatibility

This proposal is backwards compatible as long as the leader window duration and
leader schedule semantics are unchanged.

On activation, the proposer and attester schedules may be introduced as unused
derived data. Validators can derive and cache the new role assignments without
using them for block production, block validation, fork choice, or rewards.
Later proposals can define how the proposer and attester roles are consumed.

## Open Questions

1. Should validators with multiple vote accounts be sampled at the vote-account
   level, as leader scheduling does today, or aggregated by validator identity
   before proposer and attester committee construction?
2. Which committed source should define the cycle index when attestation timing
   is consensus-critical?
