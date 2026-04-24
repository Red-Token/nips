# BOLT-12 offer methods in NWC — Summary

BOLT-12 is Lightning's second-generation payment primitive. Unlike a BOLT-11 invoice (single-use, amount-fixed-at-creation), a BOLT-12 **offer** is a reusable payment handle: the receiver creates it once, publishes it, and any number of payers can invoke it. When a payer pays an offer, the wallet machinery exchanges `invoice_request` / `invoice` behind the scenes and produces a preimage — same settlement model as BOLT-11, different front door.

NIP-47 exposes offer creation, payment, status lookup, listing, and deactivation. Defined on the `patch-a-bolt12` branch of `Red-Token/nips`.

## Methods

| Method | Role | Summary |
| --- | --- | --- |
| `pay_offer` | send | Pay a BOLT-12 offer. Amount may be specified by the payer if the offer is any-amount. |
| `make_offer` | receive | Create a new BOLT-12 offer, fixed or any-amount. |
| `lookup_offer` | inspect | Get current status of an offer: active flag, payment count, total received. |
| `list_offers` | inspect | List offers created by this wallet, with pagination and an active-only filter. |
| `disable_offer` | control | Deactivate an offer so it no longer accepts payments. |

## Unit conventions

All amounts here are **msats** (Lightning native). No `_sats`-suffixed fields.

| Field | Unit |
| --- | --- |
| `pay_offer.amount` | msats (payer-specified, required if offer is any-amount) |
| `pay_offer.fees_paid` | msats |
| `make_offer.amount` | msats (omit for any-amount offer) |
| `lookup_offer.amount` | msats (present if offer is fixed-amount) |
| `lookup_offer.total_received` | msats |
| `list_offers.offers[].amount` | msats |
| `list_offers.offers[].total_received` | msats |

## Capability advertisement

Offer support is signalled by method presence in `get_info.methods`:

```jsonc
{
    "result_type": "get_info",
    "result": {
        "pubkey": "02abc...",
        "network": "mainnet",
        "block_height": 825000,
        "methods": [
            "pay_invoice", "make_invoice", "get_balance", "get_info",
            "pay_offer", "make_offer", "lookup_offer",
            "list_offers", "disable_offer"
        ]
    }
}
```

A wallet may ship a subset — for example, a payer-only wallet (can pay offers but can't create them) advertises `pay_offer` without `make_offer` / `lookup_offer` / `list_offers` / `disable_offer`.

## Example 1 — Make a fixed-amount offer

```jsonc
{
    "method": "make_offer",
    "params": {
        "amount": 10000000,
        "description": "Monthly newsletter — April 2026"
    }
}
```

```jsonc
{
    "result_type": "make_offer",
    "result": {
        "offer": "lno1qgsqvgnwgcg35z6ee2h3yczraddm72xrfua9uve2rlrm9deu7xyfzr...",
        "description": "Monthly newsletter — April 2026",
        "amount": 10000000
    }
}
```

The `offer` string is the publishable handle. 10,000,000 msats = 10,000 sats.

## Example 2 — Make an any-amount offer (tip jar pattern)

```jsonc
{
    "method": "make_offer",
    "params": {
        "description": "Tips for the stream"
    }
}
```

```jsonc
{
    "result_type": "make_offer",
    "result": {
        "offer": "lno1qgsqvgnwgcg35z6ee2h3yczraddm72xrfua9uve2rlrm9deu7xyfzr...",
        "description": "Tips for the stream"
    }
}
```

Note `amount` is absent on the response — this is the discriminator for any-amount offers. Payers will be required to specify the amount when they pay.

## Example 3 — Pay an any-amount offer with a note

```jsonc
{
    "method": "pay_offer",
    "params": {
        "offer": "lno1qgsqvgnwgcg35z6ee2h3yczraddm72xrfua9uve2rlrm9deu7xyfzr...",
        "amount": 5000000,
        "payer_note": "Great episode, thanks!"
    }
}
```

```jsonc
{
    "result_type": "pay_offer",
    "result": {
        "preimage": "2f3e4d5c6b7a8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e",
        "fees_paid": 47
    }
}
```

`payer_note` travels to the receiver's wallet and may be stored/displayed (implementation-defined). `fees_paid` is the routing fee in msats.

## Example 4 — Pay a fixed-amount offer

```jsonc
{
    "method": "pay_offer",
    "params": {
        "offer": "lno1qgsqvgnwgcg35z6ee2h3yczraddm72xrfua9uve2rlrm9deu7xyfzr..."
    }
}
```

Same response shape as Example 3. `amount` is omitted on the request because the offer carries it already. If the payer sent `amount` on a fixed-amount offer, the wallet MAY validate it matches and reject with `OTHER` if not.

## Example 5 — Inspect an offer's status

```jsonc
{
    "method": "lookup_offer",
    "params": {
        "offer": "lno1qgsqvgnwgcg35z6ee2h3yczraddm72xrfua9uve2rlrm9deu7xyfzr..."
    }
}
```

```jsonc
{
    "result_type": "lookup_offer",
    "result": {
        "offer": "lno1qgsqvgnwgcg35z6ee2h3yczraddm72xrfua9uve2rlrm9deu7xyfzr...",
        "description": "Monthly newsletter — April 2026",
        "amount": 10000000,
        "active": true,
        "num_payments_received": 23,
        "total_received": 230000000
    }
}
```

`total_received` reconciles with `num_payments_received × amount` for fixed offers; for any-amount offers it's the sum of what came in.

## Example 6 — List offers, active only

```jsonc
{
    "method": "list_offers",
    "params": {
        "active_only": true,
        "limit": 10,
        "offset": 0
    }
}
```

```jsonc
{
    "result_type": "list_offers",
    "result": {
        "offers": [
            {
                "offer": "lno1qgs...april...",
                "description": "Monthly newsletter — April 2026",
                "amount": 10000000,
                "active": true,
                "single_use": false,
                "num_payments_received": 23,
                "total_received": 230000000,
                "created_at": 1703220000
            },
            {
                "offer": "lno1qgs...tips...",
                "description": "Tips for the stream",
                "active": true,
                "single_use": false,
                "num_payments_received": 8,
                "total_received": 15400000,
                "created_at": 1703226000
            }
        ]
    }
}
```

Omit `active_only` (or set to `false`) to include disabled and expired offers too.

## Example 7 — Retire an offer

```jsonc
{
    "method": "disable_offer",
    "params": {
        "offer": "lno1qgs...april..."
    }
}
```

```jsonc
{
    "result_type": "disable_offer",
    "result": {}
}
```

After disabling, the offer's `active` flag flips to `false` on subsequent `lookup_offer` / `list_offers` calls. Inbound payment attempts are rejected by the wallet. The offer string is still parseable; it's the wallet's state that changed.

## Error cases

### `pay_offer`
- `PAYMENT_FAILED` — no route, peer refused, insufficient capacity, timeout.
- `INSUFFICIENT_BALANCE` — Lightning balance too low for `amount + fees_paid`.
- `OTHER` — malformed offer, expired offer, amount supplied on a fixed-amount offer but mismatched.

### `make_offer`
- `OTHER` — wallet refused to create (feature disabled, quota, etc.).

### `lookup_offer`, `disable_offer`
- `NOT_FOUND` — offer is not one this wallet created.

### `list_offers`
- Standard NWC errors (`RESTRICTED`, `UNAUTHORIZED`, `INTERNAL`). No method-specific errors.

## Offer payments in `list_transactions`

Offer-initiated payments flow through `list_transactions` with `payment_method: "bolt12"` (once the `payment_method` discriminator from `patch-f-rest` is in play). The offer string itself is not stored on the transaction entry by default — correlate via time window or, if implemented, by looking up the offer's `num_payments_received` counter.

## Related

- `patch-a-bolt12` — branch where these methods are specified.
- `bip321-summary.md` — a BIP-321 URI can bundle `lno=…` (a BOLT-12 offer) so `pay_bip321` can resolve to `bolt12` without explicit method choice.
- `make_bip321` accepts `{"method": "bolt12"}` in its `methods` array to include an offer in the generated URI.
