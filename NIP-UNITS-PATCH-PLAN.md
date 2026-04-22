# NIP Units Patch Plan

Proposed changes to NIP-47 (Nostr Wallet Connect) and NIP-XX (Nostr Node Control) to remove unit ambiguity on amount-bearing fields. One coherent patch touching both specs.

## Rationale

Amount units are a correctness-critical concern in a money-handling protocol. Off-by-1000 bugs can drain funds, overflow limits, or silently truncate payments. The current spec has two sources of ambiguity:

1. **Most fields are plain-named and msats by convention**, but a small set (`pay_onchain.amount`, `open_channel.amount`, `open_channel.push_amount`) are in sats. Nothing in the field name warns a reader that these break the convention. `@dukeh3/nostr-nwc` already shipped an incorrect TS comment for `open_channel.push_amount` because of this, and `ldk-controller`'s test flake that led to this audit was directly caused by the SDK treating the value as msats.

2. **On-chain quantities expressed in msats** — e.g. `get_balance.onchain_balance`, `lookup_address.total_received`, `list_addresses.addresses[].total_received`. Bitcoin's on-chain unit is the satoshi; expressing UTXO sums in msats just pads zeroes, loses no precision, and lies about the precision reality supports. Worse, it invites confusion with adjacent Lightning-denominated fields (`lightning_balance` vs `onchain_balance`).

3. **`open_channel.push_amount` is unit-bugged.** The "push" is the initial Lightning-state balance assigned to the peer when the channel opens. That's Lightning state, not on-chain state, and should be msats for consistency with every other Lightning balance in the spec. The spec currently says sats, which is both inconsistent with Lightning precision convention and a latent bug (1000× errors are indistinguishable from intended values for reasonable ranges).

## Principle

> **Every field whose value is denominated in sats MUST use the `_sats` suffix in its name. Every amount-bearing field without a `_sats` suffix is in msats.** This applies uniformly to NIP-47 and NIP-XX, to requests, responses, and notifications.

- On-chain physical values (UTXO sums, transaction outputs, funding tx amounts) take `_sats` — sat precision is the physical limit.
- Lightning values (invoices, HTLCs, channel state, fees) stay plain — msat precision is real and should not be thrown away.
- The rule enables local reasoning: a reader can tell the unit from the field name without consulting comments or method context.

## Changes to NIP-47 (47.md)

### `pay_onchain` (request)

- `amount` ⟶ `amount_sats` (value already in sats; rename only)

### `lookup_address` (response)

- `total_received` (msats) ⟶ `total_received_sats` (sats)
- `transactions[].amount` (msats) ⟶ `amount_sats` (sats) — these are on-chain txs

### `list_addresses` (response)

- `addresses[].total_received` (msats) ⟶ `total_received_sats` (sats)

### `list_transactions` (response — polymorphic entries)

Transactions may be Lightning or on-chain (discriminated by `payment_method`). Under the new rule:

- For Lightning entries (`bolt11`, `bolt12`, `keysend`): `amount` and `fees_paid` stay plain and msats.
- For on-chain entries (`payment_method: "onchain"`): entries carry `amount_sats` and `fees_paid_sats` instead. Readers keying off `payment_method` select the correct fields.

### `get_balance` (response)

- `onchain_balance` (msats, optional) ⟶ `onchain_balance_sats` (sats, optional)
- `balance` stays as-is (msats) — the mixed-nature total is intentionally kept in the smaller unit so no precision is lost when summing.
- `lightning_balance` stays as-is (msats).

### `estimate_onchain_fees` (response)

- `fees[target]` stays as-is — unit is already explicit in the value format (`sat/vB`).

### Unaffected

Every other amount field across NIP-47 (invoice amounts, offer amounts, fee values, keysend, payment notifications, hold invoices, BIP-321, estimate_routing_fees) stays plain-named and msats. These are Lightning-rooted and the NIP-47 convention is already correct.

## Changes to NIP-XX (XX.md)

### `open_channel` (request)

- `amount` (sats) ⟶ `amount_sats` (sats) — funding tx output, rename only
- `push_amount` (sats) ⟶ **`push_amount` with unit changed to msats** — this is Lightning channel state, not on-chain. **Unit fix, no rename.** Tooling and existing implementations that currently read this as sats must be updated to msats.

### `get_channel_fees` (response) and `set_channel_fees` (request)

These currently use explicit `_msat` suffixes, which deviates from NIP-47's plain-names-msats convention that this document doubles down on. `ldk-controller` already renamed them service-side in commit `b95b4a8` but the spec was never updated. Catching up:

- `base_fee_msat` ⟶ `base_fee` (unchanged semantics, still msats)
- `min_htlc_msat` ⟶ `min_htlc` (same)
- `max_htlc_msat` ⟶ `max_htlc` (same)
- `fee_rate` unchanged — unit is ppm, not an amount.

### Unaffected

All channel-balance / capacity / forwarding / route / network-stats / HTLC amount fields in NIP-XX stay plain-named and msats. These are Lightning state; convention is already correct.

## Compatibility and rollout

This patch breaks wire compatibility for the specific fields listed — that is deliberate and unavoidable: the unit bug on `push_amount` cannot be fixed compatibly, and the rename to `_sats` is the point.

### Cascade (one PR per downstream repo, each citing this spec commit)

1. **Spec** — this patch merged to `Red-Token/nips` master.
2. **`dukeh3/ldk-controller`** — rename JSON struct fields per the tables above; remove the `× 1000` conversion for `push_amount` in the control-kind handler; add a new `÷ 1000` conversion for the new `amount_sats` before calling `ldk-node.open_channel` (ldk-node still takes sats for capacity); reject `amount_sats` values that aren't whole-sat-aligned.
3. **`DarkWebDivingClub/nostr-js-nwc`** — update `OpenChannelParams` and related types: add `amount_sats` field (sats), update `push_amount` comment to msats, drop `_msat` suffix on fee fields. Same update for `onchain_balance_sats`, `total_received_sats`, on-chain transaction entries.
4. **`nostr-js-nwc-integration-test`** — update field names and unit values in helpers and tests.

Existing integration tests will catch any mistakes in the cascade. Recommend merging the spec change first so downstream PRs can cite the spec commit.

## Acceptance

- [ ] `47.md` reflects renames and unit changes as listed above.
- [ ] `XX.md` reflects renames and unit changes as listed above.
- [ ] The principle at the top of this file is referenced in both specs (or the unit rule is codified explicitly in a new section).
- [ ] Plan file committed at repo root alongside `NIP47-PATCH-PLAN.md` (if merged into master).
