# BIP-321 in NWC — Summary

BIP-321 extends the `bitcoin:` URI scheme (BIP-21) to bundle multiple payment rails — BOLT-11 invoices, BOLT-12 offers, Silent Payments, on-chain addresses — in one URI. A payer's wallet parses the URI and picks whichever method it's best equipped to use, with no round-trips to negotiate.

NIP-47 adds two NWC methods on top of BIP-321: one to **pay** a URI and one to **make** one. An optional `get_info` field advertises which rails a given wallet can produce.

Defined on the `patch-c-bip321` branch of `Red-Token/nips`.

## Methods

| Method | Role | Summary |
| --- | --- | --- |
| `pay_bip321` | send | Parse a BIP-321 URI and pay it. Wallet chooses the rail (Lightning preferred over on-chain). |
| `make_bip321` | receive | Build a BIP-321 URI bundling one or more payment rails the wallet supports. |
| `get_info.bip321_methods` | advertisement | Optional field on `get_info` listing which rails this wallet can include in a `make_bip321` output. |

## Unit conventions

| Field | Unit |
| --- | --- |
| `make_bip321.amount` | msats (Lightning-rooted interface; on-chain fallback rounds down to sats) |
| `pay_bip321.fees_paid` | msats |
| URI `amount=` output | BTC decimal (per BIP-21 inheritance) |

No `_sats`-suffixed fields. BIP-321 is treated as a Lightning-rooted surface in NWC; on-chain handling happens inside the wallet.

## Capability advertisement

The receiver wallet advertises its `make_bip321` capabilities on `get_info`:

```jsonc
{
    "result_type": "get_info",
    "result": {
        "pubkey": "02abc...",
        "network": "mainnet",
        "block_height": 825000,
        "block_hash": "0000000000000000...",
        "methods": [
            "pay_invoice", "make_invoice",
            "pay_bip321", "make_bip321",
            "get_balance", "get_info"
        ],
        "bip321_methods": [
            { "method": "bolt11" },
            { "method": "bolt12" },
            { "method": "sp" },
            { "method": "onchain", "address_types": ["p2wpkh", "p2tr"] }
        ]
    }
}
```

Rules:

- A wallet that supports neither `pay_bip321` nor `make_bip321` simply omits them from `methods` and omits `bip321_methods`.
- A wallet that can pay BIP-321 URIs but can't produce them omits `make_bip321` from `methods` and omits `bip321_methods`, but keeps `pay_bip321` in `methods`.
- Presence of `bip321_methods` implies support for `make_bip321`.

## Example 1 — Pay a URI, Lightning resolves

Request:

```jsonc
{
    "method": "pay_bip321",
    "params": {
        "uri": "bitcoin:bc1qxyz...?amount=0.0005&label=Alice&lightning=lnbc500u1p..."
    }
}
```

Response:

```jsonc
{
    "result_type": "pay_bip321",
    "result": {
        "payment_method": "bolt11",
        "preimage": "8e3a2c1f4b7d0e5a6c9b4e2d7a3f8c1b5e9a0d2c4f7b3e8a6d9c2f5b1e4a7d8c",
        "fees_paid": 120
    }
}
```

The wallet used the Lightning method bundled in the URI; `fees_paid` is the routing fee in msats. `txid` is absent because no on-chain tx was broadcast.

## Example 2 — Pay a URI, Lightning fails, on-chain fallback

Same URI as above. The wallet attempts BOLT-11 first, all routes fail, and it falls back to sending the `amount` on-chain to the URI's address.

```jsonc
{
    "result_type": "pay_bip321",
    "result": {
        "payment_method": "onchain",
        "txid": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2",
        "fees_paid": 2400000
    }
}
```

`fees_paid` here is the on-chain miner fee expressed in msats (`2_400_000` msats = 2400 sats). `preimage` is absent. The response's `payment_method` tells the caller which rail actually worked.

## Example 3 — Make a URI bundling all supported methods

Request:

```jsonc
{
    "method": "make_bip321",
    "params": {
        "amount": 50000000,
        "label": "Alice's Shop",
        "message": "Payment for order #123",
        "methods": [
            { "method": "bolt11", "expiry": 3600 },
            { "method": "bolt12" },
            { "method": "sp" },
            { "method": "onchain", "address_type": "p2tr" }
        ]
    }
}
```

Response:

```jsonc
{
    "result_type": "make_bip321",
    "result": {
        "uri": "bitcoin:bc1p...?amount=0.0005&label=Alice%27s%20Shop&message=Payment%20for%20order%20%23123&lightning=lnbc500u1p...&lno=lno1...&sp=sp1..."
    }
}
```

Note:

- `amount` on the request is **msats**. `amount=` on the URI is **BTC** — the wallet converts.
- `message` doubles as the BOLT-11 invoice `description` and the BOLT-12 offer description.
- Each sub-method can carry its own options: `expiry` for BOLT-11/BOLT-12, `address_type` for on-chain.

## Example 4 — Lightning-only wallet, no-arguments make

Request:

```jsonc
{
    "method": "make_bip321",
    "params": {
        "amount": 10000000,
        "message": "Coffee"
    }
}
```

No `methods` supplied, so the wallet includes all rails it can support. If `get_info.bip321_methods` advertised only `bolt11` and `bolt12`:

```jsonc
{
    "result_type": "make_bip321",
    "result": {
        "uri": "bitcoin:?amount=0.0001&message=Coffee&lightning=lnbc100u1p...&lno=lno1..."
    }
}
```

Notice the URI has no address after `bitcoin:` — this wallet can't include an on-chain fallback. Clients parsing this URI will only see the Lightning options.

## Error cases

### `pay_bip321`
- `PAYMENT_FAILED` — all available rails were tried and failed.
- `INSUFFICIENT_BALANCE` — funds inadequate.
- `OTHER` — URI malformed, or no supported rail in the URI, or a `req-*` parameter was not understood (BIP-321 requires rejecting the whole URI in that case).

### `make_bip321`
- `OTHER` — no rails could be generated (e.g. only `bolt11` was requested but `amount` was omitted).

## Rail selection rules

**`pay_bip321`** — the spec says the wallet SHOULD prefer Lightning methods over on-chain when multiple are present. Exact ordering within Lightning (bolt11 vs bolt12 vs sp) is left to the wallet. The `payment_method` field in the response tells the caller which rail was used.

**`make_bip321`** — the `methods` request-param is an ordered list. The wallet includes all entries it can produce; any it can't produce (e.g. `bolt11` without `amount`) are dropped silently unless that leaves the URI empty, in which case the method returns `OTHER`.

## BIP-321 in `list_transactions`

Payments made via `pay_bip321` surface in `list_transactions` using the underlying `payment_method` — `bolt11`, `bolt12`, `sp`, or `onchain`. There is no separate `"bip321"` discriminator in `list_transactions`; the URI is an input surface, not a payment type.

## Related work

- `patch-c-bip321` — branch on `Red-Token/nips` where this is specified.
- `pay_bip321` / `make_bip321` have a follow-up question around tagging the transaction with the originating URI so callers can correlate later. Not currently in the spec.
