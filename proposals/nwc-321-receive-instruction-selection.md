**NWC-321: `receive` cannot be asked for specific instructions**

## The problem

`receive` takes an amount, a description and metadata, and the wallet
selects the instructions. The spec's answer for a client that wants a
particular one is:

> A client that needs a specific instruction type uses a method for that type.

That works for a client that needs **one** instruction. It does not work
for a client that needs **several specific ones in a single URI**, which
is the case a BIP-321 URI exists for.

Concretely: "a BOLT-11 expiring in an hour, a BOLT-12 offer, and a p2tr
address, in that order of preference" cannot be requested. Calling
`make_invoice`, `make_offer` and an address method produces three separate
objects, not one URI — and nothing recombines them into one with the
payee's ordering.

A payee often does know what it wants. An invoice's expiry, an address
type, and the order the URI presents them in are all decisions the payee
is entitled to make.

## Suggested shape

One optional parameter on `receive`:

```yaml
"methods": [
    {"method": "bolt11", "expiry": 3600},
    {"method": "bolt12"},
    {"method": "onchain", "address_type": "p2tr"}
]
```

Ordered, with optional per-instruction fields (`expiry` for
bolt11/bolt12, `address_type` for onchain).

**Omitting it is today's behaviour exactly** — the wallet selects — so
this is backward-compatible in both directions: a client that does not
send it cannot tell whether a wallet implements it, and a wallet that
ignores it behaves as the spec says today.

Two rules that seem right:

- a wallet MUST NOT return an instruction type the request excluded
- a wallet MAY omit one it could not generate (BOLT-11 without an amount
  is the ordinary case) and return a URI with the rest, rather than
  failing — the call fails only if nothing could be generated

NWC-321's existing rule still binds: the URI must carry at least one
`lightning` or `lno` instruction, so a request naming only `onchain` is a
`BAD_REQUEST`.

## Why an optional extension rather than core

It is narrow, it is only meaningful for wallets that implement BIP-321
URIs at all, and a wallet ignoring the parameter conforms to NWC-321
unchanged.

## Prior implementation

We have run this shape in production-adjacent testing for some months
under a different method name, in `dln-node` (Rust, LDK) and
`nostr-nwc-ts` (TypeScript client), with a round-trip test that requests
`[bolt11, bolt12, onchain]` in one URI. We are adopting NWC-321's names
now and would rather this lived upstream than in a local extension. Happy
to open a PR if the shape looks reasonable.

---

## Addendum: `label`

Found after the above was drafted, while renaming an implementation onto
NWC-321's names.

BIP-321 carries `label` and `message` as separate parameters — `label`
names the payee, `message` says what the payment is for. `receive`'s
`description` is `message`'s job: the spec says to put it *"in each
selected instruction that supports descriptions"*. Nothing sets `label`,
so a payee cannot put its own name in the URI it hands out.

Suggested: an optional `label` on `receive`, appearing **only** in the
URI's `label=` and never as an instruction description. The two are
different claims, and conflating them tells the payer their own money's
destination twice while making the instructions disagree with the URI.

Small enough to fold into the `methods` proposal above or to take
separately.
