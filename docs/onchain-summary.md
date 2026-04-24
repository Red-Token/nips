# On-chain methods in NWC — Summary

NIP-47 was originally Lightning-only. The on-chain extension lets wallets send native Bitcoin payments, generate and inspect on-chain addresses, split the balance between Lightning and on-chain, and estimate fees — all through the same NWC envelope.

Defined on the `patch-b-onchain` branch of `Red-Token/nips`. Depends on the `_sats` suffix rule (master) to be wire-unambiguous.

## Methods

| Method | Role | Summary |
| --- | --- | --- |
| `pay_onchain` | send | Send an on-chain Bitcoin payment to an address with optional fee rate. |
| `make_new_address` | receive | Generate a fresh on-chain address of an optional script type. |
| `lookup_address` | inspect | Get total received plus the list of confirmed transactions for an address. |
| `list_addresses` | inspect | List addresses this wallet has generated via `make_new_address`. |
| `estimate_onchain_fees` | inspect | Return sat/vB fee rates for a set of confirmation targets. |
| `get_balance.onchain_balance_sats` | advertisement | Extends `get_balance` with an explicit on-chain balance field. |

## Unit conventions

| Field | Unit |
| --- | --- |
| `pay_onchain.amount_sats` | sats — on-chain tx output |
| `pay_onchain.feerate` | sat/vB |
| `lookup_address.total_received_sats` | sats — UTXO sum |
| `lookup_address.transactions[].amount_sats` | sats — tx output |
| `list_addresses.addresses[].total_received_sats` | sats |
| `estimate_onchain_fees.fees[target]` | sat/vB (floating point) |
| `get_balance.onchain_balance_sats` | sats |
| `get_balance.balance` | msats — mixed total (lightning + onchain × 1000) |

Per the global `_sats` rule: every sats-denominated field carries `_sats`; plain names mean msats. On-chain UTXOs have sat precision by construction, so no information is lost.

## Capability advertisement

On-chain capability is signalled by method presence in `get_info.methods`:

```jsonc
{
    "result_type": "get_info",
    "result": {
        "pubkey": "02abc...",
        "network": "mainnet",
        "block_height": 825000,
        "methods": [
            "pay_invoice", "make_invoice", "get_balance", "get_info",
            "pay_onchain", "make_new_address", "lookup_address",
            "list_addresses", "estimate_onchain_fees"
        ]
    }
}
```

A wallet may support any subset — for example, a receive-only wallet advertises `make_new_address`, `lookup_address`, `list_addresses` without `pay_onchain`.

## Example 1 — Mixed balance

Request:

```jsonc
{ "method": "get_balance", "params": {} }
```

Response:

```jsonc
{
    "result_type": "get_balance",
    "result": {
        "balance": 10000,
        "lightning_balance": 8000,
        "onchain_balance_sats": 2
    }
}
```

Reading: Lightning holds 8000 msats (8 sats). On-chain holds 2 sats. Total `balance` is expressed in msats: 8000 + (2 × 1000) = 10000 msats.

## Example 2 — Generate and fund an address

```jsonc
{ "method": "make_new_address", "params": { "type": "p2tr" } }
```

```jsonc
{
    "result_type": "make_new_address",
    "result": {
        "address": "bc1p5cyxn...",
        "type": "p2tr"
    }
}
```

Later, a client asks what arrived at that address:

```jsonc
{ "method": "lookup_address", "params": { "address": "bc1p5cyxn..." } }
```

```jsonc
{
    "result_type": "lookup_address",
    "result": {
        "address": "bc1p5cyxn...",
        "type": "p2tr",
        "total_received_sats": 75000,
        "transactions": [
            {
                "txid": "a1b2c3d4...",
                "amount_sats": 50000,
                "confirmations": 12,
                "timestamp": 1703224800
            },
            {
                "txid": "e5f6a7b8...",
                "amount_sats": 25000,
                "confirmations": 3,
                "timestamp": 1703225400
            }
        ]
    }
}
```

## Example 3 — Pay on-chain with explicit fee rate

```jsonc
{
    "method": "pay_onchain",
    "params": {
        "address": "bc1qxy...",
        "amount_sats": 50000,
        "feerate": 12
    }
}
```

```jsonc
{
    "result_type": "pay_onchain",
    "result": {
        "txid": "c9d2e5f8a1b4c7d0e3f6a9b2c5d8e1f4a7b0c3d6e9f2a5b8c1d4e7f0a3b6c9d2"
    }
}
```

## Example 4 — Pick a fee target

```jsonc
{ "method": "estimate_onchain_fees", "params": {} }
```

```jsonc
{
    "result_type": "estimate_onchain_fees",
    "result": {
        "fees": {
            "1":   25.0,
            "3":   12.5,
            "6":   6.0,
            "12":  3.0,
            "24":  2.0,
            "144": 1.0
        }
    }
}
```

Keys are confirmation targets (blocks). A wallet UI typically renders this as "fast / medium / slow" buttons and passes the chosen rate back to `pay_onchain.feerate`. The set of targets returned is implementation-defined — don't assume any specific key exists.

## Example 5 — List generated addresses with pagination

```jsonc
{
    "method": "list_addresses",
    "params": { "limit": 10, "offset": 0 }
}
```

```jsonc
{
    "result_type": "list_addresses",
    "result": {
        "addresses": [
            {
                "address": "bc1qaa...",
                "type": "p2wpkh",
                "total_received_sats": 75000,
                "created_at": 1703220000
            },
            {
                "address": "bc1p5cyxn...",
                "type": "p2tr",
                "total_received_sats": 0,
                "created_at": 1703226000
            }
        ]
    }
}
```

Addresses with `total_received_sats: 0` are generated-but-unused. Useful for cleanup UIs.

## Error cases

### `pay_onchain`
- `PAYMENT_FAILED` — broadcast rejected, insufficient confirmations on inputs, etc.
- `INSUFFICIENT_BALANCE` — on-chain UTXO set doesn't cover amount + fee.

### `lookup_address`
- `NOT_FOUND` — the wallet has no record of this address (it wasn't generated here, and no incoming funds observed).

### `make_new_address`, `list_addresses`, `estimate_onchain_fees`
- Standard NWC errors (`RESTRICTED`, `UNAUTHORIZED`, `INTERNAL`). No method-specific errors.

## On-chain entries in `list_transactions`

Once the `payment_method` discriminator lands (via `patch-f-rest`), on-chain transactions appear in `list_transactions` with `payment_method: "onchain"`, carrying:

- `amount_sats`, `fees_paid_sats` (not the msat-named variants)
- `txid`, `address`, `confirmations` — on-chain-only fields

This keeps unit-per-field unambiguous when Lightning and on-chain entries live in the same response array.

## Related

- `patch-b-onchain` — branch where these methods are specified.
- `bip321-summary.md` — BIP-321 is the payment-URI layer that can resolve to either on-chain (`pay_onchain`) or Lightning rails, depending on what the wallet picks.
- The `_sats` suffix rule — see the **Units** section in `47.md`.
