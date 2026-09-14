NWC-XX
======

On-chain Payments and Addresses
-------------------------------

`draft` `optional`

## Summary

This specification defines five optional Nostr Wallet Connect methods and
one response extension:

- `pay_onchain` sends an on-chain Bitcoin payment to an address.
- `make_new_address` generates a receiving address.
- `lookup_address` reports what an address has received.
- `list_addresses` lists the addresses this wallet generated.
- `estimate_onchain_fees` reports fee rates by confirmation target.
- `get_balance` gains two fields that say how much is on chain.

They are one specification because they are one capability. A wallet that
holds coins on chain can do all of this; a wallet that holds none can do
none of it. Splitting them would let a wallet declare that it can generate
an address and not say whether it can spend, which is a distinction no
client has a use for.

## Motivation

Some wallets hold on-chain funds and no Lightning channels at all: a
faucet paying from a mining wallet, a treasury service, a custodian
sweeping to cold storage. They cannot implement any Lightning method, and
today NWC gives them nothing to implement.

[NWC-321](https://github.com/nostr-wallet-connect/nwc) covers BIP-321
payment URIs, but an implementation of its `pay` method MUST support
`lightning` or `lno` instructions, so an on-chain-only wallet cannot
conform to it.

The two are complementary rather than alternatives, and the difference is
capability discovery. `pay` is polymorphic: it accepts a URI that may carry
several instruction types, and a client cannot tell from the method name
which ones the wallet will actually pay. `pay_onchain` is self-describing —
the method name *is* the capability — so it needs nothing beyond the
method-level discovery NWC core already has.

It is also a much smaller obligation. `pay` requires URI parsing,
rejection of unknown `req-` parameters, network checking, instruction
selection, and the BIP-321 `pop` / `req-pop` proof-of-payment rules. A
wallet that only ever sends coins to an address should not have to
implement any of that, particularly one whose keys are worth stealing.

## Methods

### `pay_onchain`

Sends an on-chain Bitcoin payment.

Request:
```yaml
{
    "method": "pay_onchain",
    "params": {
        "address": "bc1q...", // Bitcoin address, required
        "amount_sats": 50000, // amount in sats, required
        "feerate": 5 // fee rate in sat/vB, optional; the wallet picks a default if omitted
    }
}
```

Amounts are in **sats**, not msats, because on-chain outputs have no
sub-satoshi precision. This differs from the rest of NWC deliberately: a
msat amount here would have values that cannot be expressed on-chain.

The wallet service MUST reject an address for a different Bitcoin network.

If `feerate` is absent the wallet service selects one. If present, the
wallet service SHOULD use it and MUST NOT exceed it.

Response:
```yaml
{
    "result_type": "pay_onchain",
    "result": {
        "txid": "abc123...", // transaction id
        "fee_sats": 250 // fee paid in sats, optional if unavailable
    }
}
```

The transaction is broadcast, not confirmed. A client that needs
confirmation observes the chain or subscribes to notifications; this
response says only that the wallet service accepted and broadcast it.

Errors:

- `PAYMENT_FAILED`: The payment failed.
- `INSUFFICIENT_BALANCE`: The wallet does not have enough funds.
- `UNSUPPORTED_NETWORK`: The address is for a different Bitcoin network.
- `BAD_REQUEST`: The address or another parameter is invalid.

The wallet service can also return applicable NWC core errors.

### `make_new_address`

Generates a new on-chain Bitcoin address for receiving.

Request:

```yaml
{
    "method": "make_new_address",
    "params": {
        "type": "p2wpkh"   // "p2pkh" | "p2sh-segwit" | "p2wpkh" | "p2tr", optional
    }
}
```

Response:

```yaml
{
    "result_type": "make_new_address",
    "result": {
        "address": "bc1q...",  // the generated address
        "type": "p2wpkh"       // the type actually created
    }
}
```

The wallet picks a default when `type` is omitted, and **the response says
which** — a client that asked for nothing still learns what it got, and a
client that asked for a type the wallet does not support learns that here
rather than by comparing strings it did not send.

A wallet SHOULD return a fresh address on each call. Address reuse is a
privacy failure for the wallet's own user, and a client cannot detect it.

Errors:

- `BAD_REQUEST`: The requested address type is not one this wallet
  generates.

### `lookup_address`

Reports what an address has received.

Request:

```yaml
{
    "method": "lookup_address",
    "params": {
        "address": "bc1q..."   // required
    }
}
```

Response:

```yaml
{
    "result_type": "lookup_address",
    "result": {
        "address": "bc1q...",
        "type": "p2wpkh",
        "total_received_sats": 100000,
        "transactions": [
            {
                "txid": "abc123...",
                "amount_sats": 100000,
                "confirmations": 6,      // 0 while unconfirmed
                "timestamp": 1703225000
            }
        ]
    }
}
```

`confirmations` is `0` for a transaction in the mempool, and a client
MUST NOT treat a payment as received on the strength of a non-zero
`total_received_sats` alone — an unconfirmed output is counted and can
still disappear. What is safe to act on is a confirmation count the client
chose.

Errors:

- `NOT_FOUND`: The wallet does not watch this address.

### `list_addresses`

Lists the addresses this wallet generated through `make_new_address`.

Request:

```yaml
{
    "method": "list_addresses",
    "params": {
        "limit": 10,   // optional
        "offset": 0    // optional
    }
}
```

Response:

```yaml
{
    "result_type": "list_addresses",
    "result": {
        "addresses": [
            {
                "address": "bc1q...",
                "type": "p2wpkh",
                "total_received_sats": 100000,
                "created_at": 1703225000
            }
        ]
    }
}
```

This lists what the wallet **generated**, not every address it can spend
from. A wallet derives addresses it has never handed out, and a list
including those would leak the wallet's structure while telling the client
nothing it asked for.

### `estimate_onchain_fees`

Reports current fee rates by confirmation target.

Request:

```yaml
{
    "method": "estimate_onchain_fees",
    "params": {}
}
```

Response:

```yaml
{
    "result_type": "estimate_onchain_fees",
    "result": {
        "fees": {
            "1": 25.0,    // sat/vbyte to confirm within 1 block
            "6": 6.0,
            "144": 1.0    // ~1 day
        }
    }
}
```

`fees` maps a confirmation target in blocks to a fee rate in sat/vbyte.
**Which targets appear is the wallet's choice** and a client MUST read the
keys rather than assume a set. A wallet with one estimate returns one
entry; that is a complete answer, not a degraded one.

This is an estimate, and the same caution applies as to any fee estimate:
it is the wallet's current belief, and the rate that confirms a
transaction is the one the mempool charges when it is broadcast.

## The `get_balance` extension

NWC core's `get_balance` returns one field, `balance`, in msats. A wallet
holding funds both on chain and in channels can report a total there and
nothing more, and "how much of this can I spend on chain" has no answer.

A wallet implementing this specification MAY add two fields:

```yaml
{
    "result_type": "get_balance",
    "result": {
        "balance": 10000,             // core: the total, in msats
        "lightning_balance": 8000,    // msats, optional
        "onchain_balance_sats": 2     // sats, optional
    }
}
```

`balance` keeps its core meaning exactly — the total, in msats, being
`lightning_balance` plus `onchain_balance_sats × 1000`. **A client that
does not know this extension reads `balance` and is not wrong**, which is
the only reason adding fields to a core response is acceptable at all.

The units differ between the two added fields and that is deliberate
rather than an oversight: on-chain amounts are not expressible in msats,
and a field named `_sats` says what it holds. See the amount-units
specification, `nwc-units.md`.

A wallet MAY implement this specification's methods without these fields.
They are reporting, and a wallet that cannot separate the two balances
should omit them rather than guess.

## Relationship to other specs

- NWC core defines the request and response envelope, the error codes, and
  method discovery through the info event.
- NWC-321 defines BIP-321 payment URIs. A wallet MAY implement both. A
  wallet that implements `pay_onchain` makes no claim about Lightning, and
  a wallet that implements NWC-321's `pay` makes no claim about paying a
  bare address.
- NWC core defines `get_balance`. The two fields above extend its response
  and do not change the meaning of `balance`.
- `nwc-units.md` defines the `_sats` suffix rule, which is why
  `onchain_balance_sats` and `total_received_sats` are named as they are.
