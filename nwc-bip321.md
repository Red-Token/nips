NWC-XX
======

BIP-321 Payment URIs
--------------------

`draft` `optional`

> **This describes what we run, and upstream covers the same ground.**
> [NWC-321](https://github.com/nostr-wallet-connect/nwc) defines `pay` and
> `receive`, which are these two methods under different names and with a
> different shape. This document exists so that an implementation has
> something to cite while the move to NWC-321 is scheduled; it is **not** a
> competing proposal. See [Superseded by NWC-321](#superseded-by-nwc-321).

## Summary

This specification defines two optional Nostr Wallet Connect methods and
one `get_info` extension:

- `pay_bip321` pays a BIP-321 `bitcoin:` URI.
- `make_bip321` generates one.
- `get_info` gains `bip321_methods`, saying which instruction types the
  wallet handles.

## Motivation

A BIP-321 URI bundles several ways to pay one payee — a BOLT-11 invoice, a
BOLT-12 offer, a silent-payment address, a plain address — and lets the
payer's wallet choose. That choice is the point: the payee does not have
to know what the payer can do.

Without a method for it, a client must parse the URI itself, decide which
instruction the wallet is likely to support, and call the matching method.
It cannot know the second part, so it guesses and discovers the answer as
a failure.

## Methods

### `pay_bip321`

Pays a BIP-321 URI, choosing among the instructions it carries.

Request:

```yaml
{
    "method": "pay_bip321",
    "params": {
        "uri": "bitcoin:bc1q...?lightning=lnbc...&lno=lno1..."   // required
    }
}
```

Response:

```yaml
{
    "result_type": "pay_bip321",
    "result": {
        "payment_method": "bolt11",   // "bolt11" | "bolt12" | "sp" | "onchain"
        "preimage": "0123...",        // present for bolt11 and bolt12
        "txid": "abc123...",          // present for onchain and sp
        "fees_paid": 100              // msats, optional
    }
}
```

**`payment_method` is required in the response** because the client cannot
otherwise tell what it paid for, and the two outcomes differ in kind: a
Lightning payment is final on receipt of the preimage, while an on-chain
one is a transaction that still needs confirmations. A client that assumed
one and got the other would report success too early.

A wallet SHOULD prefer Lightning instructions over on-chain when both are
present, since the payer usually cares more about settlement speed than
the payee's preference — but this is a preference, not a rule, and a
wallet with no channels will reasonably do the opposite.

Unknown URI parameters without a `req-` prefix are ignored. **If a `req-`
parameter is not understood, the whole URI MUST be rejected** — that is
what the prefix means in BIP-321, and a wallet that ignored it would pay
an instruction the payee said was insufficient.

Errors:

- `PAYMENT_FAILED`: The payment failed.
- `INSUFFICIENT_BALANCE`: The wallet does not have enough funds.
- `BAD_REQUEST`: The URI is malformed, carries a `req-` parameter the
  wallet does not understand, or carries no instruction this wallet can
  pay.

### `make_bip321`

Generates a BIP-321 URI bundling one or more ways to be paid.

Request:

```yaml
{
    "method": "make_bip321",
    "params": {
        "amount": 50000000,               // msats, optional; required for bolt11
        "label": "Alice's Shop",          // optional, URI label only
        "message": "Order #123",          // optional, also the invoice description
        "methods": [                      // optional; all supported if omitted
            {"method": "bolt11", "expiry": 3600},
            {"method": "bolt12"},
            {"method": "sp"},
            {"method": "onchain", "address_type": "p2tr"}
        ]
    }
}
```

Response:

```yaml
{
    "result_type": "make_bip321",
    "result": {
        "uri": "bitcoin:bc1q...?amount=0.0005&label=...&lightning=lnbc..."
    }
}
```

Behaviour:

- `methods` is ordered, and the order is the payee's preference as encoded
  in the URI. Omitting it means every method the wallet can generate.
- **BOLT-11 requires `amount`.** If `amount` is omitted, `bolt11` is
  skipped rather than the call failing — a URI offering the other three is
  a useful answer, and a wallet should not refuse to be paid at all
  because one instruction could not be built.
- `message` becomes the BOLT-11 invoice description and the BOLT-12 offer
  description. `label` appears **only** in the URI's `label=` parameter —
  it names the payee, and putting it in a description would tell the payer
  their own money's destination twice while making the two instructions
  disagree.
- The URI's `amount=` is the BTC equivalent of `amount` in msats.
- At least one instruction must be generated, or the call is an error.

Errors:

- `BAD_REQUEST`: No instruction could be generated — for example `bolt11`
  alone with no `amount`.

## The `get_info` extension

A wallet implementing this specification SHOULD add `bip321_methods` to
its `get_info` response:

```yaml
{
    "result_type": "get_info",
    "result": {
        // ... core fields ...
        "bip321_methods": [
            {"method": "bolt11"},
            {"method": "bolt12"},
            {"method": "sp"},
            {"method": "onchain", "address_types": ["p2wpkh", "p2tr"]}
        ]
    }
}
```

**This is the capability discovery the method name cannot carry.**
`pay_bip321` is polymorphic: knowing a wallet implements it says nothing
about which instructions it will actually pay. Without this field a client
holding a BOLT-12-only URI cannot tell whether to bother, and finds out by
failing.

A client that does not know this extension reads the core fields and is
not wrong.

## Superseded by NWC-321

NWC-321 defines `pay` and `receive` for the same purpose. The differences
are a rename and a shape change, and **the intent is to adopt it**.

It is not adopted here for one reason, which is not a technical objection:
`pay_bip321` and `make_bip321` appear in **nine repositories** besides
this one, four of which are not ours to schedule. Adopting NWC-321 is a
coordinated rename across all of them, which is consumer-migration work
rather than specification work.

One real incompatibility is worth recording for whoever does that
migration. NWC-321's `pay` **MUST** support `lightning` or `lno`
instructions, so a wallet holding only on-chain funds cannot conform to it
at all — which is why `pay_onchain` exists as a separate specification.
A wallet migrating from `pay_bip321` to `pay` must therefore either
support Lightning or stop claiming the extension.

## Relationship to other specs

- NWC core defines the request and response envelope, the error codes, and
  method discovery through the info event.
- [NWC-321](https://github.com/nostr-wallet-connect/nwc) covers the same
  ground under different names. See above.
- The on-chain specification defines `pay_onchain`, which is what a wallet
  with no channels implements instead of either of these.
- BIP-321 defines the URI format. This adds nothing to it.
