NWC-XX
======

Payment Quotation
-----------------

`draft` `optional`

## Summary

This specification defines one optional Nostr Wallet Connect method:

- `quote_payment` reports what a payment would cost and what timelock it
  would consume, **without sending it**.

## Motivation

A client that is about to spend money should be able to say what it will
cost. NWC has `pay_invoice` and nothing that answers "how much, and how
long" beforehand.

For a wallet showing a confirmation screen that is a nicety. For anything
that must **commit to a price before paying**, it is the difference
between a protocol that works and one that guesses.

The concrete case is an exchange. A maker quoting a firm price to a
counterparty must know its own cost of delivering — the routing fee to
the destination, and the CLTV the route will consume — *before* issuing a
quotation. It cannot discover this by attempting the payment, because
attempting it is precisely what it must not do until it has been paid.

### This is a local computation, not a network query

Lightning has no way to ask the network what a route costs, and does not
need one. Every channel announces its policy in `channel_update`, so a
sender sums fees and CLTV deltas over the graph it already has. No round
trip, no counterparty.

Every implementation exposes this: LND's `QueryRoutes`, Core Lightning's
`getroute`, LDK's router. **The computation is not missing — the way to
ask your own wallet for it is.**

So this method adds no capability to Lightning. It exposes one that every
node has and NWC does not reach.

### It is an estimate, and says so

Gossip carries channel **capacities, not balances**. A path that looks
viable may fail because the liquidity sits on the wrong side, and fee
policies change between the estimate and the payment.

A wallet MUST NOT present this result as a guarantee, and a client MUST
NOT treat it as one. It is the wallet's best current belief, which is the
same basis on which the wallet would have attempted the payment anyway.

### Why not NNC's `query_routes`

NNC already has a route query: a destination and an amount, returning
routes. It is not a substitute, for two reasons.

**A wallet grant is not a node grant.** NNC and NWC are separate
protocols with separate info events and separate access grants. A
controller authorised to *spend* — `pay_invoice` and nothing else — has no
NNC access at all, and cannot ask a node-management question. Requiring
one would mean handing anybody who needs a fee estimate the ability to
inspect the node's graph, list its peers, and read its channels.

**An invoice is not a destination and an amount.** Route hints in a BOLT11
are how a payee behind a private channel is reachable at all, and they
exist only in the invoice. A query taking a pubkey would report no route
to a destination that `pay_invoice` would reach without difficulty — and
worse, would sometimes quote a route the real payment does not take.

The two answer different questions. `query_routes` asks what the graph
looks like; `quote_payment` asks what **this payment** would cost.

## Methods

### `quote_payment`

Reports the expected cost of paying an invoice, without paying it.

Request:

```yaml
{
    "method": "quote_payment",
    "params": {
        "invoice": "lnbc...",     // BOLT11, required
        "amount": 123000          // msats, required only if the invoice has no amount
    }
}
```

Response:

```yaml
{
    "result_type": "quote_payment",
    "result": {
        "amount": 123000,         // msats that would reach the destination
        "fee_msat": 1200,         // routing fee the wallet expects to pay
        "cltv_expiry_delta": 144, // blocks the route would consume
        "route_found": true
    }
}
```

`cltv_expiry_delta` is the total the route would consume, **excluding**
the invoice's own final CLTV. A caller sizing a timelock against this
payment needs both, and the second is in the invoice.

The wallet MUST NOT reserve liquidity, lock a route, or otherwise change
its state. Two calls may return different answers, and a call followed by
`pay_invoice` may cost more than the call reported.

Errors:

- `NOT_FOUND`: No route to the destination was found. This is a result,
  not a failure of the method — a wallet SHOULD return it rather than an
  internal error, since "cannot reach" is exactly what the caller asked.
- `BAD_REQUEST`: The invoice is malformed, or an amount is required and
  absent.
- `UNSUPPORTED_NETWORK`: The invoice is for a chain this wallet is not on.

The wallet service can also return applicable NWC core errors.

## Relationship to other specifications

| | |
|---|---|
| NWC core | `pay_invoice` is what this precedes. The two take the same invoice |
| [NWC-03](https://github.com/nostr-wallet-connect/nwc) | Hold invoices. A caller setting a hold invoice's final CLTV needs this method's `cltv_expiry_delta` to choose it |

## Rationale for the shape

**It takes an invoice rather than a destination and amount.** The invoice
carries the destination, the amount, the final CLTV and any route hints,
and the wallet needs all of them. A method taking a pubkey and a sum
would quote a route that the real payment might not take.

**It does not return the route.** A caller cannot use one — `pay_invoice`
does its own pathfinding, and a returned route would be advisory at best
and misleading at worst. What a caller needs is the two numbers.

**It reserves nothing.** A method that held a route or its liquidity would
let a caller tie up a wallet's capacity for free by asking repeatedly,
and a wallet cannot tell a serious enquiry from a survey.
