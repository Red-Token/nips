# A2 — Units clarification patch

Status: **on branch `patch-units`**, pending merge to master.

## What this patch does

Introduces the explicit unit rule "**plain names = msats, `_sats` suffix = sats**" into both `47.md` and `XX.md`, and applies it consistently across every amount-bearing field.

See `NIP-UNITS-PATCH-PLAN.md` at repo root for the full rationale, principle, and cascade plan.

## Spec changes

### `47.md`

- New **Units** section near the top (just before Error codes).
- `pay_onchain` req: `amount` ⟶ `amount_sats`.
- `lookup_address` resp: `total_received` (msats) ⟶ `total_received_sats` (sats); `transactions[].amount` (msats) ⟶ `amount_sats` (sats).
- `list_addresses` resp: `addresses[].total_received` (msats) ⟶ `total_received_sats` (sats).
- `list_transactions` resp: entries split — Lightning entries keep `amount` / `fees_paid` (msats); on-chain entries (`payment_method == "onchain"`) use `amount_sats` / `fees_paid_sats`.
- `get_balance` resp: `onchain_balance` (msats) ⟶ `onchain_balance_sats` (sats). `balance` stays msats (mixed total, smaller unit preserves precision). `lightning_balance` unchanged.

### `XX.md`

- New **Units** section in the Events region.
- `open_channel` req: `amount` ⟶ `amount_sats`. **`push_amount` stays named but unit changes sats ⟶ msats** — Lightning channel state.
- `get_channel_fees` / `set_channel_fees` / `get_network_channel.node{1,2}_policy`: `base_fee_msat` ⟶ `base_fee`, `min_htlc_msat` ⟶ `min_htlc`, `max_htlc_msat` ⟶ `max_htlc`. Values still msats. Spec catch-up to service rename (ldk-controller commit `b95b4a8`).

## Cascade

Downstream PRs — one per repo, each citing this spec commit:

1. `dukeh3/ldk-controller` — field renames + `push_amount` conversion fix + reject non-whole-sat `amount_sats`.
2. `DarkWebDivingClub/nostr-js-nwc` — TS type renames + comment corrections.
3. `nostr-js-nwc-integration-test` — field names + unit values in helpers and tests.

Integration test suite is the smoke test for the end-to-end cascade.

## Acceptance

- [x] `NIP-UNITS-PATCH-PLAN.md` committed.
- [x] `47.md` reflects renames and unit changes.
- [x] `XX.md` reflects renames and unit changes.
- [x] No `_msat` suffixes remain in field names (verified via grep).
- [ ] Merged to master via PR.
- [ ] Downstream cascade PRs linked from their respective task files.
