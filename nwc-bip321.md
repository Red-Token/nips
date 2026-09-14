NWC-XX
======

BIP-321 Instruction Selection and On-chain Payments
---------------------------------------------------

`draft` `optional`

> **Extends [NWC-321](https://github.com/nostr-wallet-connect/nwc), it does
> not replace it.** NWC-321 defines `pay` and `receive`. This adds the two
> things it does not cover, and defines neither method itself.

## Summary

This specification extends NWC-321's two methods:

- `receive` gains **instruction selection**: which instruction types to
  include, in what order, with per-instruction options.
- `receive` gains **`label`**, the one BIP-321 URI parameter NWC-321
  cannot set.
- `pay` gains **on-chain instruction types**, so paying a `bitcoin:`
  address from a URI can be reported.

Both are declared separately from NWC-321, and **a client that knows only
NWC-321 gets exactly what NWC-321 promises.**

## Motivation

### What `receive` cannot be told

NWC-321's `receive` takes an amount, a description and metadata, and says
*"the wallet service selects the receive instruction or instructions. A
client that needs a specific instruction type uses a method for that type."*

That answer works for a client that needs **one** instruction. It does not
work for a client that needs **several specific ones in a single URI** —
which is the case a BIP-321 URI exists for. There is no method for "a
BOLT-11 expiring in an hour, a BOLT-12 offer, and a p2tr address, in that
order of preference". Calling three methods produces three separate
objects, not one URI.

A payee often does know. An invoice with a chosen expiry, an address of a
chosen type, and an order that reflects what the payee would rather be
paid in, are all decisions the payee is entitled to make and cannot
currently express.

### What `pay` cannot report

NWC-321's `instruction_type` is `"bolt11"` or `"bolt12"`, and an
implementation of `pay` *MUST* support `lightning` or `lno`. A URI naming
only a `bitcoin:` address is a valid BIP-321 URI and a payable one, and
its payment is unrepresentable in the response.

### Why NWC-321 is right to restrict it, and what changes

**The `MUST` is a workaround for missing capability discovery.** Because
there is no way to advertise *which* instruction types a wallet handles,
the specification pins it: everyone implementing `pay` does Lightning. A
client gets a guarantee, paid for by excluding on-chain-only wallets.

**Relaxing that `MUST` to a `SHOULD` would be the wrong fix.** A wallet
could then declare `321`, support on-chain only, and a client holding a
BOLT12 offer would discover that at runtime as a failure. A declaration
that can disagree with what it answers is worse than a narrow one that
cannot.

So this specification does not relax it. It **adds the missing discovery**
— `bip321_methods` on `get_info`, below — and scopes the on-chain
capability to wallets that declare this extension. NWC-321's guarantee
survives untouched: a wallet declaring `321` still supports `lightning` or
`lno`, because this extension is declared in addition to `321` and never
instead of it.

## Discovery

A wallet implementing this specification MUST also implement NWC-321 and
MUST advertise both, separately.

It SHOULD add `bip321_methods` to its `get_info` response:

```yaml
{
    "result_type": "get_info",
    "result": {
        // ... core fields ...
        "bip321_methods": [
            {"method": "bolt11"},
            {"method": "bolt12"},
            {"method": "onchain", "address_types": ["p2wpkh", "p2tr"]}
        ]
    }
}
```

**This is the field that makes the on-chain extension safe.** `pay` is
polymorphic: knowing a wallet implements it says nothing about which
instructions it will actually pay. A client holding a BOLT12-only URI can
now tell before paying rather than by failing.

A client that does not know this extension reads NWC-321's fields and is
not wrong.

## Extending `receive`

NWC-321 defines the method. This adds one optional parameter.

Request:

```yaml
{
    "method": "receive",
    "params": {
        "amount": 123000,          // NWC-321
        "description": "string",   // NWC-321
        "metadata": {},            // NWC-321
        "methods": [               // this specification, optional
            {"method": "bolt11", "expiry": 3600},
            {"method": "bolt12"},
            {"method": "onchain", "address_type": "p2tr"}
        ]
    }
}
```

`methods` is **ordered**, and the order is the payee's preference as
encoded in the URI.

Each entry has a required `method` and optional per-instruction fields:

| Field | Applies to | |
|---|---|---|
| `expiry` | `bolt11`, `bolt12` | seconds; the wallet's default when absent |
| `address_type` | `onchain` | `p2pkh`, `p2sh-segwit`, `p2wpkh`, `p2tr`; the wallet's default when absent |

**Omitting `methods` is NWC-321's behaviour exactly**: the wallet selects.
That is what makes this an extension rather than a change — a client that
does not send the parameter cannot tell this specification is implemented.

A wallet MUST NOT return an instruction type the request excluded. It MAY
omit one the request asked for and could not generate — **BOLT-11 without
an amount is the ordinary case**, since BOLT-11 requires one — and a URI
carrying the rest is a useful answer rather than an error. The call fails
only if **no** instruction could be generated.

NWC-321's own rule still binds: the returned URI MUST contain at least one
`lightning` or `lno` instruction. A `methods` array naming only `onchain`
is therefore a `BAD_REQUEST` for a wallet declaring `321`.

Errors:

- `BAD_REQUEST`: `methods` is empty, names an unknown type, names only
  instruction types NWC-321 does not permit alone, or carries a
  per-instruction option this wallet cannot honour.

### `label`

BIP-321 URIs carry `label` and `message` as **separate** parameters:
`label` names the payee, `message` says what the payment is for.

NWC-321's `receive` takes `description` and puts it *"in each selected
instruction that supports descriptions"* — which is `message`'s job. There
is no way to set `label`, so a payee cannot put its own name in the URI it
hands out.

```yaml
{
    "method": "receive",
    "params": {
        "description": "Order #123",   // NWC-321: the instruction descriptions
        "label": "Alice's Shop"        // this specification: the URI's label=
    }
}
```

`label` MUST appear **only** in the URI's `label=` parameter and MUST NOT
be used as an instruction description. The two are different claims —
putting the payee's name where the payment's purpose belongs tells the
payer their own money's destination twice and makes the instructions
disagree with the URI.

## Extending `pay`

NWC-321 defines the method. This widens one response field.

`instruction_type` MAY additionally be:

| | |
|---|---|
| `onchain` | a `bitcoin:` address instruction was paid |
| `sp` | a silent-payment instruction was paid |

When either is reported, `txid` is REQUIRED and `preimage` MUST be absent
— there is no preimage. This is the reason the field is worth extending
rather than leaving a client to infer: **the two outcomes differ in kind.**
A Lightning payment is final on the preimage; an on-chain one is a
transaction that still needs confirmations, and a client that assumed one
and got the other would report success too early.

A wallet MUST NOT select an on-chain instruction when the URI carries a
`lightning` or `lno` instruction it can pay, unless the client's own
policy says otherwise. NWC-321's preference is preserved.

Errors: as NWC-321.

## Relationship to other specs

- [NWC-321](https://github.com/nostr-wallet-connect/nwc) defines `pay`,
  `receive`, the BIP-321 processing rules, the `req-` handling and the
  response envelope. **Required**, and none of it is restated here.
- NWC core defines `get_info`, which `bip321_methods` extends.
- The on-chain payments specification defines `pay_onchain` for a wallet
  with no channels at all — which cannot implement NWC-321's `pay`, and so
  cannot implement this either.
- Both additions here are **written as proposals to NWC-321 and have not
  been sent** — see [`proposals/`](proposals/). If they are sent and
  adopted there, this document is withdrawn rather than maintained.
