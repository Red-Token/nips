# NIP-47 Patch Plan

Proposed changes to NIP-47 (Nostr Wallet Connect), organized as independent reviewable patches. Each patch builds on the previous but can be reviewed and discussed separately.

## Patch A: BOLT-12

Add BOLT-12 offer support to NWC.

- `pay_offer` — pay a BOLT-12 offer with optional amount and payer_note
- `make_offer` — create an offer with amount and description
- `lookup_offer` — check offer status (active, num_payments_received, total_received)
- `list_offers` — list offers with active_only filter, pagination
- `disable_offer` — deactivate an offer so it no longer accepts payments
- Update kind 13194 info event content string with BOLT-12 methods

## Patch B: On-chain

Add on-chain bitcoin operations to NWC.

- `pay_onchain` — send an on-chain payment with optional feerate
- `make_new_address` — generate a new address with optional type param
- `lookup_address` — look up address info with transaction history
- `list_addresses` — list generated addresses with pagination
- `estimate_onchain_fees` — return fee estimates by confirmation target
- `get_balance` — add optional `lightning_balance` and `onchain_balance` fields
- Add on-chain fields (`txid`, `address`, `confirmations`) to `list_transactions` response and `payment_received`/`payment_sent` notifications
- Update kind 13194 info event content string with on-chain methods

## Patch C: BIP-321

Add BIP-321 `bitcoin:` URI support to NWC — unified payment and receive across methods.

- `pay_bip321` — pay a BIP-321 URI; wallet parses and selects the best available payment method (lightning preferred over on-chain)
- `make_bip321` — generate a BIP-321 URI bundling one or more payment methods (bolt11, bolt12, onchain, sp) with per-method options (expiry, address_type)
- Add `bip321_methods` field to `get_info` response — advertises which payment methods the wallet supports for `make_bip321`
- Update kind 13194 info event content string with BIP-321 methods

## Patch D: Subscriptions

Add a notification model and subscription mechanism.

- New "Notification Model" section defining default behavior: creator gets notified about outcomes of operations it initiates
- Notification linkage: which methods trigger which notifications (e.g. `make_invoice` → `payment_received`, `pay_invoice` → `payment_sent`)
- Opt-out: `"notify": false` parameter suppresses notification for a specific request
- `subscribe_notifications` — subscribe to receive notifications for all events, not just self-initiated ones (monitoring/accounting use case)

## Patch E: Formalities

Editorial and consistency improvements.

- Add "wallet application" term — distinguishes the actual wallet/node software (LND, CLN, LDK) from the "wallet service" (Nostr translation layer)
- RFC-style wording: `"can"` → `"MAY"` in 2 places (expiration tag, connection URI keys)
- `created_at` made optional (`"optional if unknown"`) in 7 response/notification structures (make_invoice, lookup_invoice, list_transactions, make_hold_invoice, payment_received, payment_sent, hold_invoice_accepted)

## Patch F: Rest

Remaining method additions.

- `estimate_routing_fees` — estimate fee and CLTV delta for routing a payment to a destination
- `payment_method` field — add `"bolt11"` / `"bolt12"` / `"onchain"` / `"keysend"` discriminator to `list_transactions` request (filter), `list_transactions` response entries, `payment_received` notification, and `payment_sent` notification
- `list_invoices` — list invoices by state (pending/settled/expired) with pagination; invoice management complement to `list_transactions`
