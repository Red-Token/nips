NWC-XX
======

BOLT12 Offer Management
-----------------------

`draft` `optional`

> **Extends [NWC-12](https://github.com/nostr-wallet-connect/nwc), it does
> not replace it.** NWC-12 defines `make_offer` and the `bolt12` payment
> type. This adds the three things it does not cover.

## Summary

This specification defines three optional Nostr Wallet Connect methods:

- `pay_offer` pays a BOLT12 offer.
- `list_offers` lists the offers this wallet created.
- `disable_offer` stops one accepting payments.

## Motivation

NWC-12 is **receive-side**. It creates offers, keeps them payable, and
defines how a payment made through one is looked up. It says nothing about
paying somebody else's offer, and nothing about the offers a wallet has
already created.

Those are the operator's questions rather than the payer's, and they do
not fit inside `make_offer`:

- **Paying.** NWC-321's `pay` can carry an `lno` instruction, but only
  inside a BIP-321 URI, and a client holding a bare `lno1...` string has
  nothing to call. Wrapping it in a URI to pay it is a workaround, not an
  interface.
- **Listing.** An offer is reusable, so "which of my offers are live and
  what have they taken" has no answer in a payment-lookup method. NWC-12
  says it plainly: *an offer is a receive target, not a payment record.*
- **Retiring.** An offer that should stop working needs the wallet to stop
  issuing invoices against it. Nothing else can do it — the string is
  already published.

### What was here before

An earlier draft of this document defined `make_offer` and `lookup_offer`
as well. Both are withdrawn.

`make_offer` duplicated NWC-12's, which had been published in the interval
between our survey and our draft, and NWC-12's is the better definition —
it carries `offer_id`, `issuer`, `single_use` and an absolute expiry, and
it requires the wallet to keep every unexpired offer payable while the
user-facing wallet is offline.

`lookup_offer` is withdrawn because `list_offers` below returns the same
per-offer statistics, and a payment is looked up with NWC-09's
`lookup_payment`. Two methods for one question is how implementations
drift.

## Dependencies

- [NWC-12](https://github.com/nostr-wallet-connect/nwc) for `make_offer`,
  `offer_id`, and the `bolt12` payment type.
- [NWC-09](https://github.com/nostr-wallet-connect/nwc) for
  `lookup_payment`, which NWC-12 requires.

**Offers are identified by NWC-12's `offer_id`** — the 32-byte BOLT12
offer ID as lowercase hex — and not by the encoded offer string. A wallet
implementing this specification MUST implement NWC-12.

## Methods

### `pay_offer`

Pays a BOLT12 offer.

Request:

```yaml
{
    "method": "pay_offer",
    "params": {
        "offer": "lno1...",        // required, the encoded offer
        "amount": 123000,          // msats, required if the offer has no fixed amount
        "payer_note": "string"     // optional, shown to the payee
    }
}
```

Response:

```yaml
{
    "result_type": "pay_offer",
    "result": {
        "transaction_id": "wallet-scoped-id",
        "preimage": "0123456789abcdef...",
        "fees_paid": 1000            // msats, optional
    }
}
```

**The offer, not an `offer_id`.** Paying is the one operation where the
wallet did not create the object: a payer holds a string somebody else
published, and has no `offer_id` for it until it has parsed it.

`transaction_id` is NWC-09's, so the payer can reconcile the payment with
`lookup_payment` afterwards. Returning it here is what makes this method
composable with the rest of NWC-12's world rather than a dead end.

Paying an offer involves a round trip the client does not see — the wallet
fetches an invoice from the payee and pays that — so this can fail in a way
`pay_invoice` cannot: the payee may be unreachable or may decline to issue
an invoice at all. A wallet SHOULD distinguish that from a routing
failure, because the remedies differ. A payer can retry a route; nobody
can retry a refusal.

Errors:

- `PAYMENT_FAILED`: The payment failed, including the payee declining or
  failing to issue an invoice.
- `INSUFFICIENT_BALANCE`: The wallet does not have enough funds.
- `BAD_REQUEST`: The offer is malformed, or an amount is required and
  absent.

### `list_offers`

Lists offers this wallet created.

Request:

```yaml
{
    "method": "list_offers",
    "params": {
        "active_only": true,       // optional, default false
        "limit": 10,               // optional
        "offset": 0                // optional
    }
}
```

Response:

```yaml
{
    "result_type": "list_offers",
    "result": {
        "offers": [
            {
                "offer_id": "0123456789abcdef...",
                "offer": "lno1...",
                "description": "string",        // optional
                "issuer": "example.com",        // optional
                "amount": 123000,               // msats, null for variable-amount
                "active": true,
                "single_use": false,
                "num_payments_received": 5,
                "total_received": 615000,       // msats
                "created_at": 1703225000,
                "expires_at": 1703311400        // optional
            }
        ]
    }
}
```

The per-offer fields follow NWC-12's `make_offer` response, plus three
this specification adds: `active`, `num_payments_received` and
`total_received`.

`active` is `false` for an offer this wallet has disabled or whose expiry
has passed. **It is not a statement about the network**: a disabled offer
is still a valid string a payer may hold, and what stops it working is
this wallet declining to issue invoices against it.

`active_only` defaults to **false** so the unfiltered call is the complete
one. A disabled offer that received payments is exactly what an operator
reconciling accounts needs to see, and a default that hid it would make
the honest question the harder one to ask.

`num_payments_received` and `total_received` count **settled** payments.
An offer with an accepted-but-unsettled payment against it has not
received it yet.

### `disable_offer`

Stops an offer accepting payments.

Request:

```yaml
{
    "method": "disable_offer",
    "params": {
        "offer_id": "0123456789abcdef..."   // required
    }
}
```

Response:

```yaml
{
    "result_type": "disable_offer",
    "result": {}
}
```

**This is not revocation and cannot be.** The offer string is already
published and payers holding it will keep trying. What this does is make
the wallet stop issuing invoices against it, so those attempts fail. A
client presenting this to a user should say so — "stop accepting", not
"delete" — because a user who believes an offer is gone may reuse the
context it was published in.

A wallet MUST NOT settle a new payment against a disabled offer. It MAY
settle one whose invoice was issued before the offer was disabled, since
the payer committed funds against a promise the wallet had already made.

Disabling is idempotent: disabling a disabled offer succeeds.

Errors:

- `NOT_FOUND`: This wallet did not create the offer, or it is not visible
  to this connection.

## Relationship to other specs

- [NWC-12](https://github.com/nostr-wallet-connect/nwc) defines
  `make_offer`, `offer_id` and the `bolt12` payment type. Required.
- [NWC-09](https://github.com/nostr-wallet-connect/nwc) defines
  `lookup_payment`, which is how a payment made through an offer is
  reconciled — including one made by `pay_offer`.
- [NWC-321](https://github.com/nostr-wallet-connect/nwc) can pay an `lno`
  instruction inside a BIP-321 URI. `pay_offer` takes the offer itself.
- NWC core defines the request and response envelope, the error codes, and
  method discovery through the info event.
