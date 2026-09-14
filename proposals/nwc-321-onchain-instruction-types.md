**NWC-321: `pay` cannot report that it paid an on-chain instruction**

## The problem

`instruction_type` is `"bolt11"` or `"bolt12"`, and an implementation of
`pay` MUST support `lightning` or `lno`.

A BIP-321 URI naming only a `bitcoin:` address is a valid URI and a
payable one. Its payment has no representation in the response — the
wallet either refuses a URI it could pay, or pays it and cannot say what
it did.

The response already carries `txid`, described as an on-chain transaction
identifier, which suggests this was half-anticipated.

## Why the current `MUST` is right, and what we are *not* proposing

We think the `MUST` is doing real work: **it is standing in for missing
capability discovery.** Because there is no way to advertise which
instruction types a wallet handles, the spec pins it, and a client gets a
guarantee it can rely on.

So we are **not** proposing to relax it to a `SHOULD`. That would let a
wallet declare `321`, support on-chain only, and a client holding a BOLT12
offer would find out at runtime as a failure. A declaration that can
disagree with what it answers is worse than a narrow one that cannot.

## Suggested shape

Add the discovery, and scope the widening to wallets that declare it.

**1. `bip321_methods` on `get_info`:**

```yaml
"bip321_methods": [
    {"method": "bolt11"},
    {"method": "bolt12"},
    {"method": "onchain", "address_types": ["p2wpkh", "p2tr"]}
]
```

**2. `instruction_type` MAY additionally be `onchain` or `sp`**, for
wallets that advertise those. When either is reported, `txid` is required
and `preimage` is absent.

That last point is the reason this is worth a field rather than left to
inference: **the two outcomes differ in kind.** A Lightning payment is
final on the preimage; an on-chain one is a transaction that still needs
confirmations. A client that assumed one and got the other reports success
too early.

The existing guarantee survives: a wallet declaring `321` still supports
`lightning` or `lno`. Lightning-first selection stays as specified.

## Why an optional extension rather than core

It only concerns wallets implementing BIP-321 URIs, and a client that does
not read `bip321_methods` sees NWC-321 exactly as it is today.

## Prior implementation

Run under a different method name in `dln-node` (Rust, LDK), which pays
`lightning`, `lno` and address instructions from one URI and reports which
it used. We are adopting NWC-321's names now; this is the one thing we
cannot express under them, and we would rather propose it than keep a
local extension. Happy to open a PR.

## Note

If the answer is "on-chain instructions are out of scope for this spec,
by design", that is a clear answer and we will document it as the reason a
separate on-chain method exists. We would rather ask than assume.
