NWC-XX
======

Amount Units
------------

`draft` `optional`

## Summary

This specification defines a naming convention, not a method:

- A field denominated in **sats** carries a `_sats` suffix.
- A field without one is in **msats**.

## Motivation

NWC carries amounts in two units and the field name is the only place the
unit can live. A wallet and a client that disagree about which one a field
uses are out by a factor of a thousand, and nothing in the protocol
notices: both values are plausible integers, both serialise, both arrive.

This has cost us real time. `open_channel`'s `push_amount` was read as
sats by one implementation and msats by another, and the symptom was an
intermittently failing integration test rather than anything pointing at a
unit.

The rule is worth stating precisely because the failure it prevents is
silent, and because a convention that is only followed most of the time is
worse than none — a reader who has seen it hold twice will assume it holds
the third time.

## The rule

> Every field whose value is denominated in sats MUST carry the `_sats`
> suffix in its name. Every amount-bearing field **without** a `_sats`
> suffix is in msats.

It applies uniformly to requests, responses and notifications.

Where the two could both apply, the object decides:

- **On-chain physical values** — UTXO sums, transaction output amounts,
  on-chain balances — take `_sats`, because sats are what the chain
  counts. There is no such thing as a millisat output.
- **Lightning values** — invoice amounts, HTLC amounts, channel state,
  routing fees — stay plain, because msats are what Lightning counts, and
  a fee smaller than one sat is ordinary rather than exceptional.

So `push_amount` is **msats**: it is channel state, not an on-chain
output, even though the funding transaction that creates the channel is
on-chain. The unit follows what the field describes, not what paid for it.

## Applying it

A field that already carries the right unit but the wrong name is a
**rename**, and a field that carries the wrong unit is a **behaviour
change**. The two look identical in a diff and must not be treated alike:
a rename breaks callers loudly, and a unit flip breaks them silently, for
a thousand times the value they expected.

An implementation adopting this rule SHOULD list which of its fields are
which before changing any of them.

## Relationship to other specs

- NWC core defines the fields this names. The rule adds no field and
  changes no wire format beyond the names themselves.
- NIP-XX (node control) applies the same rule, and the two are meant to
  stay identical — a reader who learns it once should not have to learn it
  again.
