# ADR 004: Storage Archival and Nullifier Lifetime

- **Status:** Accepted
- **Date:** 2026-08-31
- **Context:** Design for issue #254 (Nullifier entries can be archived — a claim could be replayed after TTL expiry).

## Context

In Soroban, persistent storage entries are archived when their TTL lapses.

Previously, the double-claim fence in `Contract::claim` relied on standalone persistent storage entries:
```rust
let nullifier_key = DataKey::Nullifier(circle_id, nullifier_hash.clone());
if env.storage().persistent().has(&nullifier_key) {
    panic_with_error!(&env, Error::AlreadyClaimed);
}
```

Soroban persistent entries have their TTL extended at write time to `LEDGER_EXTEND_TO = 500_000` ledgers (~1 month at 5s/ledger). Nullifiers are write-once per claim and were never accessed or extended again.

In contrast, the `Circle` entry (`DataKey::Circle(circle_id)`) is continuously re-extended on every `fund` and `claim` call. If a circle remains active or is extended, but individual `DataKey::Nullifier` entries lapse and are archived, `env.storage().persistent().has(&nullifier_key)` would return `false`. This opened a vulnerability where a member could replay a previously used claim/nullifier in a subsequent round after its TTL expired.

## Options Analyzed

### Option (a) — Re-extend every circle's nullifier TTLs on each claim
- On every claim, iterate through all previously stored nullifiers for the circle and extend their TTLs.
- **Drawback**: Unbounded storage lookups and CPU/TTL extension work as the number of claims grows; fails to scale.

### Option (b) — Store nullifiers as a bounded `Vec<Fr>` inside the `Circle` struct (Chosen)
- Add `pub nullifiers: Vec<Fr>` to `pub struct Circle`.
- Read and check `circle.nullifiers.contains(&nullifier_hash)` directly within `claim` and `has_claimed`.
- Store nullifiers directly inside `Circle` at `DataKey::Circle(circle_id)`.
- **Advantages**:
  - Nullifiers inherit the `Circle` entry's continuously-extended TTL lifecycle. As long as the `Circle` entry is live or re-extended, all nullifiers registered for that circle remain live.
  - For a fixed-size ROSCA, the number of nullifiers in a circle is bounded by design (`size` members per cycle).
  - In-memory `Vec::contains()` on a small `soroban_sdk::Vec` is O(N) where N ≤ circle size, which is cheap and fits comfortably within CPU instruction limits.

### Option (c) — Declare circles time-bounded
- Require circles to complete within the 500,000 ledger TTL window and accept replay risks for expired circles.
- **Drawback**: Leaves residual vulnerability if a circle spans more than ~1 month.

## Decision

Adopt **Option (b)**. Nullifiers are embedded within `Circle.nullifiers`.

## Consequences

- Nullifiers never archive independently of the `Circle` entry.
- Double-claim fences survive arbitrary ledger advancement as long as the circle exists.
- State size per `Circle` increases by `32 * N` bytes, naturally bounded by circle size.
