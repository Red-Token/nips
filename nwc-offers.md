NWC-XX
======

BOLT-12 Offers
--------------

`draft` `optional`

## Summary

This specification defines five optional Nostr Wallet Connect methods:

- `pay_offer` pays a BOLT-12 offer.
- `make_offer` creates one.
- `lookup_offer` reports what an offer has received.
- `list_offers` lists the offers this wallet created.
- `disable_offer` stops one accepting payments.

## Motivation

NWC core is BOLT-11: `make_invoice` produces an invoice for one payment of
one amount, and `lookup_invoice` reports whether that payment arrived. A
client that wants a **reusable** way to be paid has nothing to ask for.

BOLT-12 offers are that. An offer is a long-lived instruction to pay,
which any number of payers can fetch an invoice against, and the payee
does not have to be online when a payer decides to use it. That is a
different object from an invoice, with a different lifecycle, and it needs
its own methods rather than a flag on the invoice ones.

### Why not extend `make_invoice`

Because the two differ in what they promise. An invoice names one payment
and expires; an offer names a willingness to be paid and is retired. The
words that make sense for one are wrong for the other — an offer has no
`settled_at`, and asking whether an offer is "paid" is a category error
when the answer is "five times, so far".

Overloading `make_invoice` with a mode flag would mean a response whose
fields are conditionally meaningless, and a client discovering which
by reading the flag it sent.

### An offer is not a secret

Anything a payer can fetch an invoice against is public by intent. So
`list_offers` and `lookup_offer` report **usage** — how many payments, how
much received — because that is the part the wallet's owner cannot see by
looking at the offer itself, and the part no payer can see at all.

## Methods

### `make_offer`

Creates a BOLT-12 offer.

Request:

```yaml
{
    "method": "make_offer",
    "params": {
        "amount": 123,             // msats; omit for an any-amount offer
        "description": "string"    // required
    }
}
```

Response:

```yaml
{
    "result_type": "make_offer",
    "result": {
        "offer": "lno1...",        // the encoded offer
        "description": "string",
        "amount": 123              // msats, present only if fixed
    }
}
```

**Omitting `amount` is meaningful, not a default.** An any-amount offer
lets the payer choose, which is what a donation address or a tip jar is.
A wallet that cannot issue one MUST return `BAD_REQUEST` rather than
substituting an amount of its own.

`description` is required because it is what the payer sees before paying
and the only thing distinguishing two otherwise identical offers in
`list_offers`.

### `pay_offer`

Pays a BOLT-12 offer.

Request:

```yaml
{
    "method": "pay_offer",
    "params": {
        "offer": "lno1...",        // required
        "amount": 123,             // msats, required if the offer has no fixed amount
        "payer_note": "string"     // optional, shown to the payee
    }
}
```

Response:

```yaml
{
    "result_type": "pay_offer",
    "result": {
        "preimage": "0123456789abcdef...",
        "fees_paid": 123           // msats, optional
    }
}
```

Paying an offer involves a round trip the client does not see: the wallet
fetches an invoice from the payee and pays that. So this method can fail
in a way `pay_invoice` cannot — the payee may be unreachable, or may
decline to issue an invoice at all — and a wallet SHOULD distinguish that
from a routing failure, because the remedies differ. A payer can retry a
route; nobody can retry a refusal.

Errors:

- `PAYMENT_FAILED`: The payment failed — timeout, no route, insufficient
  capacity, or the payee did not issue an invoice.
- `INSUFFICIENT_BALANCE`: The wallet does not have enough funds.
- `BAD_REQUEST`: The offer is malformed, or an amount is required and
  absent.

### `lookup_offer`

Reports an offer's status and what it has received.

Request:

```yaml
{
    "method": "lookup_offer",
    "params": {
        "offer": "lno1..."         // required
    }
}
```

Response:

```yaml
{
    "result_type": "lookup_offer",
    "result": {
        "offer": "lno1...",
        "description": "string",
        "amount": 123,                  // msats, present only if fixed
        "active": true,
        "num_payments_received": 5,
        "total_received": 615           // msats
    }
}
```

`active` is `false` for an offer this wallet has disabled or that has
expired. **It is not a statement about the network**: a disabled offer is
still a valid string a payer may hold, and what makes it stop working is
this wallet declining to issue invoices against it.

Errors:

- `NOT_FOUND`: This wallet did not create the offer. A wallet MUST NOT
  report on an offer it does not own, since it has no way to know what
  that offer received.

### `list_offers`

Lists offers created by this wallet.

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
                "offer": "lno1...",
                "description": "string",
                "amount": 123,                  // msats, present only if fixed
                "active": true,
                "single_use": false,
                "num_payments_received": 5,
                "total_received": 615,          // msats
                "created_at": 1703225000
            }
        ]
    }
}
```

`active_only` defaults to **false** so that the unfiltered call is the
complete one. A disabled offer that has received payments is exactly what
an operator reconciling accounts needs to see, and a default that hid it
would make the honest question the harder one to ask.

### `disable_offer`

Stops an offer accepting payments.

Request:

```yaml
{
    "method": "disable_offer",
    "params": {
        "offer": "lno1..."         // required
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

**This is not revocation and cannot be.** The offer string is already out
in the world, and payers holding it will keep trying. What this does is
make the wallet stop issuing invoices against it, so those attempts fail.
A client presenting this to a user should say so — "stop accepting", not
"delete" — because a user who believes an offer is gone may reuse the
context it was published in.

Disabling is idempotent: disabling a disabled offer succeeds.

Errors:

- `NOT_FOUND`: This wallet did not create the offer.

## Relationship to other specs

- NWC core defines the request and response envelope, the error codes, and
  method discovery through the info event.
- NWC core's `make_invoice` and `lookup_invoice` are the BOLT-11
  equivalents. A wallet MAY implement either set, both, or neither.
- BOLT-12 defines the offer format itself. This specification defines how
  a client asks a wallet to use one, and adds nothing to BOLT-12.
