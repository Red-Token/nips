NIP-XX
======

Nostr Node Control (NNC)
------------------------

`draft` `optional`

## Rationale

[NIP-47](47.md) (Nostr Wallet Connect) defines a protocol for clients to access a remote Lightning wallet — paying invoices, creating invoices, checking balances, and listing transactions. These are **wallet operations**: spending and receiving sats.

However, Lightning node owners and administrators need a separate set of operations to **manage their node**: opening and closing channels, connecting to peers, adjusting fee policies, inspecting the network graph, and monitoring routing activity. These operations do not belong in a wallet protocol because they change the node's topology and configuration rather than moving funds.

NNC (Nostr Node Control) fills this gap. It is a companion protocol to NWC, using the same architectural patterns (connection URI, encrypted request/response over relays, replaceable info event) but with a distinct set of commands for node administration.

## Terms

* **client**: Nostr app on any platform that wants to manage a Lightning node.
* **user**: The person using the **client**, typically the node owner or an authorized administrator.
* **node service**: Nostr app running alongside the Lightning node (or on the node itself) that translates NNC requests into node API calls. This is analogous to the NIP-47 **wallet service**.
* **owner**: The node operator who controls access grants.

## Theory of Operation

The connection flow mirrors NIP-47:

1. The **node service** publishes an NNC info event (kind `13198`) to its relay(s). The **owner** configures the **node service** with their pubkey and publishes access grants (kind `30078`) for authorized controllers.

2. The **user** imports the connection URI into their **client** (QR code, deeplink, or paste). The **client** fetches the info event to discover which NNC methods the **node service** supports.

3. When the **user** wants to perform a node operation (e.g. open a channel), the **client** creates an NNC request event (kind `23198`), encrypts the payload with [NIP-44](44.md), and publishes it to the relay(s) from the connection URI.

4. The **node service** decrypts the request, executes the operation against the Lightning node, and publishes an encrypted NNC response event (kind `23199`).

5. For methods with deferred outcomes (e.g. channel open/close), the **node service** sends an encrypted NNC notification event (kind `23200`) back to the requesting **client** when the operation completes. The **client** MAY set `"notify": false` to suppress this notification.

## Events

There are four event kinds:

- `NNC info event`: **13198** (replaceable)
- `NNC request`: **23198** (ephemeral)
- `NNC response`: **23199** (ephemeral)
- `NNC notification event`: **23200** (ephemeral)

### Info Event (kind 13198)

The info event is a replaceable event published by the **node service** on the relay to indicate which NNC capabilities it supports.

The content SHOULD be a plaintext string with the supported methods space-separated:

```
list_channels open_channel close_channel list_peers connect_peer disconnect_peer get_channel_fees set_channel_fees get_forwarding_history get_pending_htlcs estimate_route_fee query_routes list_network_nodes get_network_stats get_network_node get_network_channel subscribe_notifications
```

### Request Event (kind 23198)

The request event SHOULD contain one `p` tag with the public key of the **node service**.

Optionally, a request can have an `expiration` tag with a unix timestamp in seconds. If the request is received after this timestamp, it should be ignored.

The content is encrypted with [NIP-44](44.md) and is a JSON object:

```jsonc
{
    "method": "list_channels", // method name, string
    "params": {               // params, object
        // method-specific parameters
    }
}
```

### Response Event (kind 23199)

The response event SHOULD contain one `p` tag with the public key of the **user** and an `e` tag with the id of the request event it is responding to.

The content is encrypted with [NIP-44](44.md) and is a JSON object:

```jsonc
{
    "result_type": "list_channels", // indicates the structure of the result field
    "error": {                      // object, non-null in case of error
        "code": "UNAUTHORIZED",     // string error code, see below
        "message": "human readable error message"
    },
    "result": {   // result, object. null in case of error.
        // method-specific result data
    }
}
```

The `result_type` field MUST contain the name of the method that this event is responding to.
The `error` field MUST contain a `message` field with a human readable error message and a `code` field with the error code if the command was not successful.
If the command was successful, the `error` field must be null.

### Notification Event (kind 23200)

The notification event is sent only in response to a request that had `"notify": true`. It SHOULD contain one `p` tag with the public key of the requesting **client** and an `e` tag with the id of the original request event.

The content is encrypted with [NIP-44](44.md) and is a JSON object:

```jsonc
{
    "notification_type": "channel_opened", // indicates the structure of the notification field
    "notification": {
        // notification-specific data
    }
}
```

### Error Codes

- `RATE_LIMITED`: The client is sending commands too fast. It should retry in a few seconds.
- `NOT_IMPLEMENTED`: The command is not known or is intentionally not implemented.
- `RESTRICTED`: This public key is not allowed to do this operation.
- `UNAUTHORIZED`: This public key has no node connected.
- `QUOTA_EXCEEDED`: The controller has exceeded its spending quota.
- `NOT_FOUND`: The requested resource (channel, peer, node, etc.) was not found.
- `INTERNAL`: An internal error.
- `OTHER`: Other error.

## NNC Connection URI

The **client** discovers the **node service** by scanning a QR code, handling a deeplink, or pasting in a URI.

The connection URI uses the protocol `nostr+nodecontrol://` and base path the hex-encoded `pubkey` of the **node service** with the following query string parameters:

- `relay` Required. URL of the relay where the **node service** is connected and will be listening for events. May be more than one.

The **client** uses its own key pair to sign events and encrypt payloads. The **owner** must publish an access grant (kind `30078`) for the **client**'s pubkey before it can issue requests.

### Example connection string

```sh
nostr+nodecontrol://b889ff5b1513b641e2a139f661a661364979c5beee91842f8f0ef42ab558e9d4?relay=wss%3A%2F%2Frelay.damus.io
```

## Commands

### Channel Management

#### `list_channels`

Description: Lists the node's channels with their current status and balances.

Request:
```jsonc
{
    "method": "list_channels",
    "params": {}
}
```

Response:
```jsonc
{
    "result_type": "list_channels",
    "result": {
        "channels": [
            {
                "id": "abc123",                // channel ID (implementation-specific)
                "short_channel_id": "800000x1x0", // short channel ID, optional
                "peer_pubkey": "02abc...",      // remote peer's pubkey
                "state": "active",             // "active", "inactive", "pending_open", "pending_close", "force_closing"
                "is_private": false,           // whether the channel is private
                "local_balance": 500000,       // local balance in msats
                "remote_balance": 500000,      // remote balance in msats
                "capacity": 1000000,           // total channel capacity in msats
                "funding_txid": "abc123...",    // funding transaction ID
                "funding_output_index": 0      // funding transaction output index
            }
        ]
    }
}
```

#### `open_channel`

Description: Opens a channel to a peer. If not already connected, the **node service** SHOULD connect to the peer first.

Request:
```jsonc
{
    "method": "open_channel",
    "params": {
        "pubkey": "02abc...",             // peer's pubkey, required
        "amount": 1000000,               // channel capacity in sats, required
        "push_amount": 0,                // amount to push to peer in sats, optional, default 0
        "private": false,                // whether the channel should be private, optional, default false
        "host": "10.0.0.1:9735",         // peer's host:port, optional (for auto-connect)
        "close_address": "bc1q...",      // cooperative close address, optional
        "notify": true                   // send notification when the channel is confirmed, optional, default true
    }
}
```

Response:
```jsonc
{
    "result_type": "open_channel",
    "result": {
        "funding_txid": "abc123..."      // funding transaction ID
    }
}
```

Errors:
- `CHANNEL_FAILED`: The channel could not be opened. This may be due to insufficient funds, peer refusing, or similar.

#### `close_channel`

Description: Closes a channel. Defaults to a cooperative close; set `force` to true for a force close.

Request:
```jsonc
{
    "method": "close_channel",
    "params": {
        "id": "abc123",                  // channel ID, required
        "force": false,                  // force close, optional, default false
        "close_address": "bc1q...",      // address to send funds to, optional
        "notify": true                   // send notification when the close is confirmed, optional, default true
    }
}
```

Response:
```jsonc
{
    "result_type": "close_channel",
    "result": {
        "closing_txid": "def456..."      // closing transaction ID
    }
}
```

Errors:
- `NOT_FOUND`: The channel was not found.
- `CHANNEL_FAILED`: The channel could not be closed.

### Peer Management

#### `list_peers`

Description: Lists the node's connected peers.

Request:
```jsonc
{
    "method": "list_peers",
    "params": {}
}
```

Response:
```jsonc
{
    "result_type": "list_peers",
    "result": {
        "peers": [
            {
                "pubkey": "02abc...",         // peer's pubkey
                "address": "10.0.0.1:9735",  // peer's address
                "connected": true,           // whether currently connected
                "alias": "ACINQ",            // peer's alias, optional
                "num_channels": 2            // number of channels with this peer
            }
        ]
    }
}
```

#### `connect_peer`

Description: Connects to a peer.

Request:
```jsonc
{
    "method": "connect_peer",
    "params": {
        "pubkey": "02abc...",              // peer's pubkey, required
        "host": "10.0.0.1:9735"           // peer's host:port, required
    }
}
```

Response:
```jsonc
{
    "result_type": "connect_peer",
    "result": {}
}
```

Errors:
- `CONNECTION_FAILED`: Could not connect to the peer.

#### `disconnect_peer`

Description: Disconnects from a peer.

Request:
```jsonc
{
    "method": "disconnect_peer",
    "params": {
        "pubkey": "02abc..."               // peer's pubkey, required
    }
}
```

Response:
```jsonc
{
    "result_type": "disconnect_peer",
    "result": {}
}
```

Errors:
- `NOT_FOUND`: The peer was not found.

### Fees & Routing

#### `get_channel_fees`

Description: Gets the fee policies for the node's channels.

Request:
```jsonc
{
    "method": "get_channel_fees",
    "params": {
        "id": "abc123"                   // channel ID, optional (omit for all channels)
    }
}
```

Response:
```jsonc
{
    "result_type": "get_channel_fees",
    "result": {
        "fees": [
            {
                "id": "abc123",              // channel ID
                "short_channel_id": "800000x1x0", // short channel ID, optional
                "peer_pubkey": "02abc...",    // remote peer's pubkey
                "base_fee_msat": 1000,       // base fee in msats
                "fee_rate": 1,               // proportional fee rate in millionths (ppm)
                "min_htlc_msat": 1000,       // minimum HTLC size in msats, optional
                "max_htlc_msat": 500000000   // maximum HTLC size in msats, optional
            }
        ]
    }
}
```

#### `set_channel_fees`

Description: Updates the fee policy for a channel or all channels.

Request:
```jsonc
{
    "method": "set_channel_fees",
    "params": {
        "id": "abc123",                  // channel ID, optional (omit to apply to all channels)
        "base_fee_msat": 1000,           // base fee in msats, optional
        "fee_rate": 1,                   // proportional fee rate in ppm, optional
        "min_htlc_msat": 1000,           // minimum HTLC size in msats, optional
        "max_htlc_msat": 500000000       // maximum HTLC size in msats, optional
    }
}
```

Response:
```jsonc
{
    "result_type": "set_channel_fees",
    "result": {}
}
```

Errors:
- `NOT_FOUND`: The channel was not found.

#### `get_forwarding_history`

Description: Lists forwarded payments (routing events).

Request:
```jsonc
{
    "method": "get_forwarding_history",
    "params": {
        "from": 1693876973,              // starting timestamp in seconds (inclusive), optional
        "until": 1703225078,             // ending timestamp in seconds (inclusive), optional
        "limit": 50,                     // maximum number of forwards to return, optional
        "offset": 0                      // offset of the first forward to return, optional
    }
}
```

Response:
```jsonc
{
    "result_type": "get_forwarding_history",
    "result": {
        "forwards": [
            {
                "incoming_channel_id": "abc123",   // incoming channel
                "outgoing_channel_id": "def456",   // outgoing channel
                "incoming_amount": 10000,          // incoming amount in msats
                "outgoing_amount": 9990,           // outgoing amount in msats
                "fee_earned": 10,                  // fee earned in msats
                "settled_at": 1703225000           // timestamp in seconds
            }
        ]
    }
}
```

#### `get_pending_htlcs`

Description: Lists in-flight HTLCs across channels.

Request:
```jsonc
{
    "method": "get_pending_htlcs",
    "params": {}
}
```

Response:
```jsonc
{
    "result_type": "get_pending_htlcs",
    "result": {
        "htlcs": [
            {
                "channel_id": "abc123",        // channel this HTLC is on
                "direction": "incoming",       // "incoming" or "outgoing"
                "amount": 10000,               // HTLC amount in msats
                "hash_lock": "abcdef...",      // payment hash
                "expiry_height": 800100        // block height at which this HTLC expires
            }
        ]
    }
}
```

#### `estimate_route_fee`

Description: Estimates the fee for routing a payment to a destination.

Request:
```jsonc
{
    "method": "estimate_route_fee",
    "params": {
        "destination": "02abc...",       // destination pubkey, required
        "amount": 100000                 // payment amount in msats, required
    }
}
```

Response:
```jsonc
{
    "result_type": "estimate_route_fee",
    "result": {
        "fee": 150,                      // estimated fee in msats
        "time_lock_delay": 40            // estimated CLTV delta
    }
}
```

Errors:
- `NOT_FOUND`: No route found to the destination.

#### `query_routes`

Description: Finds possible routes to a destination.

Request:
```jsonc
{
    "method": "query_routes",
    "params": {
        "destination": "02abc...",       // destination pubkey, required
        "amount": 100000,               // payment amount in msats, required
        "max_routes": 3                  // maximum number of routes to return, optional, default 1
    }
}
```

Response:
```jsonc
{
    "result_type": "query_routes",
    "result": {
        "routes": [
            {
                "total_fee": 150,            // total fee for this route in msats
                "total_time_lock": 40,       // total CLTV delta
                "hops": [
                    {
                        "pubkey": "02abc...",       // hop node pubkey
                        "short_channel_id": "800000x1x0", // channel used
                        "fee": 50,                  // fee for this hop in msats
                        "expiry": 20               // CLTV delta for this hop
                    }
                ]
            }
        ]
    }
}
```

Errors:
- `NOT_FOUND`: No route found to the destination.

### Network Graph

#### `list_network_nodes`

Description: Lists nodes in the public network graph.

Request:
```jsonc
{
    "method": "list_network_nodes",
    "params": {
        "limit": 50,                     // maximum number of nodes to return, optional
        "offset": 0                      // offset of the first node to return, optional
    }
}
```

Response:
```jsonc
{
    "result_type": "list_network_nodes",
    "result": {
        "nodes": [
            {
                "pubkey": "02abc...",          // node pubkey
                "alias": "ACINQ",             // node alias, optional
                "color": "#3399ff",           // node color, optional
                "num_channels": 150,          // number of public channels
                "total_capacity": 50000000000, // total capacity across channels in msats
                "addresses": [                // network addresses, optional
                    "10.0.0.1:9735"
                ],
                "last_update": 1703225000     // last gossip update timestamp
            }
        ]
    }
}
```

#### `get_network_stats`

Description: Gets aggregate statistics about the network graph.

Request:
```jsonc
{
    "method": "get_network_stats",
    "params": {}
}
```

Response:
```jsonc
{
    "result_type": "get_network_stats",
    "result": {
        "num_nodes": 15000,                   // total number of nodes
        "num_channels": 65000,                // total number of channels
        "total_capacity": 500000000000000,    // total network capacity in msats
        "avg_channel_size": 7692307692,       // average channel size in msats
        "max_channel_size": 1000000000000     // largest channel size in msats
    }
}
```

#### `get_network_node`

Description: Gets information about a specific node in the network graph.

Request:
```jsonc
{
    "method": "get_network_node",
    "params": {
        "pubkey": "02abc..."               // node pubkey, required
    }
}
```

Response:
```jsonc
{
    "result_type": "get_network_node",
    "result": {
        "pubkey": "02abc...",              // node pubkey
        "alias": "ACINQ",                 // node alias, optional
        "color": "#3399ff",               // node color, optional
        "num_channels": 150,              // number of public channels
        "total_capacity": 50000000000,    // total capacity in msats
        "addresses": [                    // network addresses, optional
            "10.0.0.1:9735"
        ],
        "last_update": 1703225000,        // last gossip update timestamp
        "features": {}                    // feature bits, optional
    }
}
```

Errors:
- `NOT_FOUND`: The node was not found in the graph.

#### `get_network_channel`

Description: Gets information about a specific channel in the network graph.

Request:
```jsonc
{
    "method": "get_network_channel",
    "params": {
        "short_channel_id": "800000x1x0"   // short channel ID, required
    }
}
```

Response:
```jsonc
{
    "result_type": "get_network_channel",
    "result": {
        "short_channel_id": "800000x1x0",     // short channel ID
        "capacity": 1000000000,                // channel capacity in msats
        "node1_pubkey": "02abc...",            // first node's pubkey
        "node2_pubkey": "03def...",            // second node's pubkey
        "node1_policy": {                      // first node's routing policy, optional
            "base_fee_msat": 1000,
            "fee_rate": 1,
            "min_htlc_msat": 1000,
            "max_htlc_msat": 500000000,
            "time_lock_delta": 40,
            "disabled": false,
            "last_update": 1703225000
        },
        "node2_policy": {                      // second node's routing policy, optional
            "base_fee_msat": 500,
            "fee_rate": 2,
            "min_htlc_msat": 1000,
            "max_htlc_msat": 500000000,
            "time_lock_delta": 40,
            "disabled": false,
            "last_update": 1703224000
        }
    }
}
```

Errors:
- `NOT_FOUND`: The channel was not found in the graph.

## Notifications

### Notification Model

**Default behavior**: A controller is notified about deferred outcomes of operations it initiates. This mirrors the NIP-47 notification model.

- `open_channel` → `channel_opened` when the channel is confirmed on-chain
- `close_channel` → `channel_closed` when the close is confirmed on-chain

**Opt-out**: Set `"notify": false` in the request to suppress the notification for that specific operation.

**Subscription**: Controllers that need to monitor _all_ channel events (e.g. a dashboard that didn't initiate the open/close) should use the `subscribe_notifications` method.

#### `subscribe_notifications`

Request:
```jsonc
{
    "method": "subscribe_notifications",
    "params": {
        "types": ["channel_opened", "channel_closed"], // notification types to subscribe to, required
    }
}
```

To unsubscribe, call with an empty types array:
```jsonc
{
    "method": "subscribe_notifications",
    "params": {
        "types": [], // empty array unsubscribes from all
    }
}
```

Response:
```jsonc
{
    "result_type": "subscribe_notifications",
    "result": {}
}
```

Payment-related notifications (`payment_received`, `payment_sent`) belong to NIP-47 (NWC).

### `channel_opened`

Description: Sent when a channel from an `open_channel` request has been confirmed on-chain and is now active.

Notification:
```jsonc
{
    "notification_type": "channel_opened",
    "notification": {
        "id": "abc123",                    // channel ID
        "short_channel_id": "800000x1x0",  // short channel ID, optional
        "peer_pubkey": "02abc...",          // remote peer's pubkey
        "capacity": 1000000,               // total channel capacity in msats
        "local_balance": 1000000,          // local balance in msats
        "remote_balance": 0,               // remote balance in msats
        "funding_txid": "abc123...",        // funding transaction ID
        "is_private": false                // whether the channel is private
    }
}
```

### `channel_closed`

Description: Sent when a channel from a `close_channel` request has been confirmed on-chain.

Notification:
```jsonc
{
    "notification_type": "channel_closed",
    "notification": {
        "id": "abc123",                    // channel ID
        "short_channel_id": "800000x1x0",  // short channel ID, optional
        "peer_pubkey": "02abc...",          // remote peer's pubkey
        "capacity": 1000000,               // total channel capacity in msats
        "closing_txid": "def456...",        // closing transaction ID
        "close_type": "cooperative"        // "cooperative", "force_local", "force_remote", "breach"
    }
}
```

## Encryption

All NNC request, response, and notification payloads MUST be encrypted using [NIP-44](44.md). The encryption uses the **client**'s private key and the **node service**'s public key.

## Access Control

This section defines the access control model for both NWC (NIP-47) and NNC methods. A single **node service** MAY serve both protocols and SHOULD use this unified model.

### Access Grants (kind 30078)

The **owner** publishes parameterized replaceable events of kind `30078` to grant access to controllers.

- The event `pubkey` is the **owner**'s pubkey.
- The event `content` is a JSON-encoded `UsageProfile`.
- The event includes a `d` tag whose value is `node_pubkey:controller_pubkey`.
- The event `tags` MUST include a `p` tag with the **node service**'s pubkey so it receives the event.
- The event `tags` MAY include auxiliary metadata (e.g., label, scope, or policy identifiers).
- The event is signed by the **owner** and published to the relay(s) from the connection URI.
- Subsequent updates use the same kind, pubkey, and `d` tag; the newest `created_at` replaces earlier grants.

### UsageProfile JSON

The `UsageProfile` defines per-controller permissions and limits. All numeric values are unsigned integers.

```json
{
  "methods": {
    "get_info": {},
    "get_balance": {
      "rate": {
        "rate_per_micro": 10,
        "max_capacity": 1000
      }
    },
    "pay_invoice": {}
  },
  "control": {
    "connect_peer": {},
    "open_channel": {},
    "close_channel": {},
    "list_channels": {}
  },
  "quota": {
    "rate_per_micro": 1,
    "max_capacity": 1000000
  }
}
```

Fields:

- `methods` (object, optional): Map of NWC method name to a `MethodAccessRule`.
    - Missing or empty `methods` means no NWC permissions are granted.
    - The special key `"ALL"` grants access to all NWC methods. Rate limits on the `"ALL"` entry apply to every method.
    - A method MUST be explicitly present (or `"ALL"` granted) to be allowed.
- `control` (object, optional): Map of NNC method name to a `MethodAccessRule`.
    - Missing or empty `control` means no NNC permissions are granted.
    - The special key `"ALL"` grants access to all NNC methods. Rate limits on the `"ALL"` entry apply to every method.
    - A method MUST be explicitly present (or `"ALL"` granted) to be allowed.
- `methods.<method>.rate` (object, optional): Per-method rate limit. If missing, no rate limit is applied.
    - `rate_per_micro` (u64, optional): Tokens refilled per microsecond. Default `0`.
    - `max_capacity` (u64, optional): Maximum token capacity. Default `u64::MAX`.
- `control.<method>.rate` (object, optional): Per-control-method rate limit. If missing, no rate limit is applied.
    - `rate_per_micro` (u64, optional): Tokens refilled per microsecond. Default `0`.
    - `max_capacity` (u64, optional): Maximum token capacity. Default `u64::MAX`.
- `quota` (object, optional): Controller-wide spending quota. If missing, no quota is applied.
    - `rate_per_micro` (u64, optional): Quota refill per microsecond. Default `0`.
    - `max_capacity` (u64, optional): Maximum quota capacity. Default `u64::MAX`.

Defaults:

- Missing numeric fields use their defaults (`rate_per_micro = 0`, `max_capacity = u64::MAX`).
- Missing optional objects are treated as absent limits.

### Revocation

To revoke a controller's access:

1. Publish a grant with empty `methods` `{}` and empty `control` `{}`. This immediately removes all permissions.
2. Publish a kind `5` deletion event referencing the grant event id. This cleans up the grant from relays.

Step 1 ensures immediate revocation regardless of whether the relay processes the deletion. Step 2 is housekeeping — relay deletion propagation is not guaranteed, but the empty grant from step 1 already denies all access.

### Request Handling Pipeline

1. **Decode**: Decrypt the event (NIP-44) and parse the JSON request.
2. **Resolve Method**: Check that the method name is known. If not, return `NOT_IMPLEMENTED`.
3. **Authorize**: Check whether the controller pubkey is permitted to call the method. Check `methods` (NWC) or `control` (NNC) in the `UsageProfile`.
4. **Enforce Limits**: Apply rate and quota checks using the controller's access state. Limits are evaluated without mutating state.
5. **Validate**: Validate the request parameters for the given method. If validation fails, return an error response.
6. **Execute**: Dispatch the request to the node.
7. **Commit Usage**: If execution succeeds, apply the rate/quota usage to state. Usage is committed only after the request is accepted for execution.

### Authorization Steps

1. Resolve the caller's latest kind `30078` grant for `d = node_pubkey:controller_pubkey`. If missing, deny with `UNAUTHORIZED`.
2. Parse the grant content as `UsageProfile`. If parsing fails, deny with `UNAUTHORIZED`.
3. Look up the requested method in the applicable map — `methods` for NWC requests, `control` for NNC requests. If the map is missing, empty, or the requested method is not present and `"ALL"` is not present, deny with `RESTRICTED`.
4. Check per-method rate limit (from the `rate` field on the method entry). If the rate is exceeded, deny with `RATE_LIMITED`.
5. Check `quota`. If the method involves spending and the quota is exceeded, deny with `QUOTA_EXCEEDED`.
6. Grant access.

## Relationship to NIP-47

NIP-47 (NWC) and NIP-XX (NNC) are complementary protocols with a clear boundary:

| | NIP-47 (NWC) | NIP-XX (NNC) |
|---|---|---|
| **Purpose** | Wallet operations | Node administration |
| **Audience** | Apps, merchants, services | Node owners, admins |
| **Operations** | Pay, receive, check balance | Channels, peers, fees, graph |
| **Info kind** | 13194 | 13198 |
| **Request kind** | 23194 | 23198 |
| **Response kind** | 23195 | 23199 |
| **Notification kind** | 23196 | 23200 |
| **URI scheme** | `nostr+walletconnect://` | `nostr+nodecontrol://` |

A node dashboard application would typically use **both** protocols: NWC for payment operations (pay invoice, create invoice, check balance) and NNC for node management (open channel, set fees, monitor routing).

The two protocols use separate connection URIs and separate key pairs, allowing independent authorization. A user can grant an application NWC access without granting NNC access, and vice versa.

## Example Flow

1. The node operator configures their **node service** and scans or copies the `nostr+nodecontrol://` URI into their **client** application.
2. The **client** fetches the NNC info event (kind `13198`) from the relay to learn which NNC commands are available.
3. The user wants to open a channel. The **client** creates an `open_channel` request with `"notify": true`, encrypts it with NIP-44, and publishes kind `23198` to the relay.
4. The **node service** decrypts the request, calls the Lightning node's channel open API, and publishes an encrypted kind `23199` response with the funding transaction ID.
5. Once the channel is confirmed on-chain, the **node service** sends a `channel_opened` notification (kind `23200`) back to the **client** that made the request.
