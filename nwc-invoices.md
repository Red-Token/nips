NWC-XX
======

Invoice Listing
---------------

`draft` `optional`

## Summary

This specification defines one optional Nostr Wallet Connect method:

- `list_invoices` lists invoices this wallet created, with their current
  state.

## Motivation

### What this is not

**NWC-05's `list_transactions` is payment history. This is not that**, and
the difference is the whole reason it exists.

A payment history lists what happened: money moved, here is the record. An
invoice that was never paid did not happen, so it is not there. Neither is
one still waiting, nor one that expired unpaid.

Those are precisely the invoices somebody managing invoices needs to see.
A shop wants to know which orders are outstanding; an operator wants to
know which invoices expired so they can be reissued; a client showing a
list of what it asked for cannot build that list from a record of what
arrived.

So the two methods answer different questions:

| | `list_transactions` (NWC-05) | `list_invoices` |
|---|---|---|
| lists | payments, in and out | invoices this wallet created |
| includes unpaid | no — nothing happened | **yes, that is the point** |
| includes expired | no | **yes** |
| keyed on | the payment | the invoice |

A wallet MAY implement both, and one is not derivable from the other.

### Why not a filter on `list_transactions`

Because the filter would have to invent rows. A pending invoice has no
transaction to filter — there is nothing in the history to select — so
`list_transactions` with `state: "pending"` would have to synthesise
entries for events that did not occur. A method that sometimes returns
things that happened and sometimes things that did not is worse than two
methods.

## Methods

### `list_invoices`

Lists invoices created by this wallet.

Request:

```yaml
{
    "method": "list_invoices",
    "params": {
        "from": 1693876973,    // unix seconds, inclusive, optional
        "until": 1703225078,   // unix seconds, inclusive, optional
        "limit": 10,           // optional
        "offset": 0,           // optional
        "state": "pending"     // "pending" | "settled" | "expired", optional
    }
}
```

Response:

```yaml
{
    "result_type": "list_invoices",
    "result": {
        "invoices": [
            {
                "invoice": "lnbc50n1...",
                "description": "string",          // optional
                "payment_hash": "string",
                "amount": 123,                    // msats
                "state": "pending",               // "pending" | "settled" | "expired"
                "preimage": "string",             // present only when settled
                "created_at": 1693876973,
                "expires_at": 1693880573,
                "settled_at": 1693877000          // present only when settled
            }
        ]
    }
}
```

`from` and `until` filter on **`created_at`**, not on settlement. An
invoice created yesterday and paid today belongs to yesterday, because the
question this method answers is "what did I ask for", and the thing being
listed came into existence when it was created.

### The three states

`expired` and `pending` are distinguished by the wallet's clock, not the
client's, and a wallet MUST report `expired` for an invoice past
`expires_at` rather than leaving it `pending` for the client to work out.
A client cannot do that reliably: it may have a wrong clock, and it has no
way to know whether the wallet would still accept a late payment.

**A settled invoice stays settled past its expiry.** Expiry bounds when a
payment may be *initiated*, and an invoice that was paid is not retroactively
expired by the clock. The states are ordered by what happened, not by time.

`preimage` and `settled_at` are present exactly when `state` is `settled`.
A client MAY use either as the test, and a wallet MUST NOT return one
without the other.

Errors:

- `BAD_REQUEST`: `state` is not one of the three values.

The wallet service can also return applicable NWC core errors.

## Relationship to other specs

- NWC core defines `make_invoice` and `lookup_invoice`. This lists what
  the first created; `lookup_invoice` answers about one of them by hash.
- [NWC-05](https://github.com/nostr-wallet-connect/nwc) defines
  `list_transactions`, which is payment history. See **What this is not**.
- The offers specification defines `list_offers`, which is the same idea
  for a different object: what this wallet created, and what became of it.
