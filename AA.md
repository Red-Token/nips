NIP-AA
======

Nostr Communities
----------------------

`draft` `optional`

This NIP extends the definition of `kind:0` events to include support for communities.

## Abstract

This NIP describes how `kind:0` can be extended to provide more information, about the user. This includes defining a
new type of user, classed as a community as opposed to the traditional individual user. Community users are encouraged
to publish and distribute NIP-65 events to advertise on what relays they share there information.

A community user can be used by individual users to coordinate different kinds of information. Imagine a group of
fishing interested individuals that want to run a server where they organise events using nip52, share videos using
nip71, and have a group chat using nip 29. How can the attract new community members? Well they create a community user
and publish the user on different capabilities of the community and make a nice description. They then publish this user
to different public relays using an `kind:0`, they then publish a `kind:65` to match. Now users interested in fishing
can find there relay and join.

## The "n" tag

The n tag is used to advertise capabilities that a user is expected to support to be able to participate in the
community.

`["n", <nostrid>, <optional future parameters> ]`

Where:

* `<nostrid>` represent the supported nip, used by the user / community.

## The "c" tag

`["c", <class> ]`

* `<class>` represent the type of user, either 'individual' or 'community'. If no c tag is present its assumed that we
  are talking about an individual.

## Example

In this example the user represents a community where the users share and discuss videos around fishing.

```json
{
  "content": {
    "name": "Big Fish",
    "about": "A fishing community in Main"
  },
  "tags": [
    ["c","community"],
    ["n","nip29"],
    ["n","nip52"],
    ["n","nip71"]
  ]
}
```
