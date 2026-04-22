# A1 — Merge `nip47-list-methods` into master

Status: **done** — fast-forwarded master to `2d559f4`.

## Context

The `nip47-list-methods` branch added four NIP-47 methods that `ldk-controller` (service) and `@dukeh3/nostr-nwc` (SDK) had already implemented. The spec had been trailing for 11 days. This task caught it up.

## Scope

Single fast-forward merge of `origin/nip47-list-methods` into `origin/master`. No conflicts — merge-base was the previous master tip `74a8a3b`.

## Adds to `47.md`

- `list_offers` — list offers with `active_only` filter + pagination
- `list_invoices` — list invoices by state (pending / settled / expired) + pagination
- `list_addresses` — list generated on-chain addresses + pagination
- `disable_offer` — deactivate an offer

## Follow-up

Task **A2** (NIP-UNITS-PATCH-PLAN) will re-examine `list_addresses` naming under the new `_sats` suffix rule — `total_received` on on-chain addresses becomes `total_received_sats`.

## Acceptance

- [x] Fast-forward clean (`git merge --ff-only`).
- [x] Master contains all four method sections.
- [x] Branch `nip47-list-methods` deleted on remote (done alongside this commit).
