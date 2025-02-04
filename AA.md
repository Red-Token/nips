NIP-AA
======

Nostr Communities
----------------------

`draft` `optional`

This NIP extends the definition of `kind:0` events to include support for communities.

## Abstract

This NIP describes how `kind:0` can be extended to provide more information, about the user. Community users are encouraged
To create 

## Background

## The "n" tag

`["n", <nostrid>, <optional future parameters> ]`

Where:

 * `<nostrid>` represent the supported nip, used by the user / community.

## The "c" tag

`["c", <class> ]`

* `<class>` represent the type of user, either 'individual' or 'community'. If no c tag is present its assumed that we are talking about an individual.

## Example 

In this example the user represents a community where the users share and discuss videos around fishing.

```json
{
  "c": "community",
  "n": "nip29",
  "n": "nip71"
}
```
