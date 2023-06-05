# SNIP-52 - Private Push Notifications

Allows clients to receive notifications when certain events within a smart contract affect them. For example: token transfers, direct messaging, turn-based gaming, announcements, and so on.

* [Introduction](#introduction)
  * [Motivation](#motiviation)
  * [Overview](#overview)
    * [How it works](#how-it-works-high-level)
    * [Example flow](#example-flow)
* [Concepts]
  * [Notification ID](#notification-id)
  * [Channels](#channels)
  * [Notification Seed](#notification-seed)
  * [Internal Contract Secret](#internal-contract-secret)
  * [Counter](#counter)
  * [Notification Data](#notification-data)
* Queries and Methods
  * [ListChannels Query](#listchannels-query)
  * [ChannelInfo Query](#channelinfo-query)
  * [UpdateSeed Method](#updateseed-method)
* [Notification ID Algorithm](#notification-id-algorithm)
* [Privacy Considerations](#privacy-considerations)
  * [Decoys](#decoys)

# Introduction

## Motivation 

Wallets and dApps currently resort to a polling-based approach in order to notice changes to a user's private state within a contract.

For example, a dApp might periodically query a set of SNIP-2x token contracts to discover a new incoming transfer.

However, this approach of querying contracts every so often is rather inefficient and can create unwanted load on query nodes. Additionally, there is no clear best practice for determining an optimal polling rate.


## Overview

This document presents a technique that allows clients to receive notifications for specific future events (e.g., "next incoming transfer of token X", or "once it is my turn in chess") such that all information about the notification (e.g., its recipient, supplemental data, and whether or not an event actually ocurred) remains private, encrypted, and unaccessible to outside observers.

Keep in mind, the only time smart contracts mutate state is during execution, which can only occur during a network transaction. For example, Alice's "token X" balance can only ever change in response to a transaction that executes some method on that contract (e.g., "Bob transfers 100 token X to Alice"). It is only in response to these executions that a contract would emit a notification to notify some recipient(s) of an event that affected them (e.g., "hey Alice, someone just transferred tokens to your account").


### How it works (high-level)

First, let's review the basic components that make this possible:

1. The existing [Tendermint event stack](https://docs.tendermint.com/v0.34/tendermint-core/subscription.html) is a [publish-subscribe](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) model that allows nodes to transmit network events directly to subscribed clients. It does so using JSONRPC over WebSockets.

2. Leveraging the [`add_attribute_plaintext(...)`](https://github.com/scrtlabs/SecretNetwork/blob/ddb9b67a23a2eb5f64076fa3e0a836d186a02b63/cosmwasm/contracts/v1/compute-tests/test-compute-contract/src/contract.rs#L60C1-L62) API method, Secret Contracts can add custom key-value attributes to the transaction's log.

3. Clients coordinate with smart contracts to determine globally unique, single-use "Notification IDs" which represent specific future events. When one of those specific events occur, its Notication ID is added to the transaction log as a custom attribute. Since the contract and recipient are the only parties aware of a Notification ID's significance, the event in that transaction's log is what discreetly notifies the recipient, effectively creating a private push-based notification service.


### Example flow

Let's walk through a simple example, where client Alice wants to be notified next time she receives a transfer of token X into her account.

For simplicity, this example does not make use of custom notification data.

> NOTE: This example uses fake data for brevity. Actual seeds and IDs are much longer.

1. Alice queries the token X contract to get the unique Notification ID for her next incoming transfer:
    ```json
    {
      "next_notification_id": {
        "channel": "transfers"
      }
    }
    ```

    The contract responds:
    ```json
    {
      "next_notification_id": {
        "channel": "transfers",
        "seed": "ecc7f60418aa",
        "next_id": "ZjZjYzVhYjU4",
        "as_of_block": "1131420"
      }
    }
    ```

    Alice now has a globally unique Notification ID the contract will use next time someone transfers tokens to her account.

    Furthermore, Alice now has a seed she can use to derive future Notification IDs offline for subsequent transfer events (i.e., without having to query this contract again)

2. Before, after, or simultaneously with step 1, Alice subscribes to transaction events:
    ```json
    {
      "jsonrpc": "2.0",
      "id": "0",
      "method": "subscribe",
      "params": {
        "query": "tm.event='Tx'"
      }
    }
    ```

    Alice will now receive a JSONRPC message for each new transaction event from the chain, from which she can search for her unique Notification ID.
    
    Alternatively, if she trusts the WebSocket server won't record her activity, she can use a filter in the `query` field of her subscription message, e.g., `tm.event='Tx' AND wasm.ZjZjYzVhYjU4 EXISTS` .

3. Some time later, Bob executes a SNIP-20 transfer of token X to Alice's account:
    ```json
    {
      "transfer": {
        "recipient": "Alice",
        "amount": "100",
      }
    }
    ```

    The contract derives the next transfer Notification ID for the recipient (Alice) and adds it as a custom attribute to the transaction log.
    ```json
    {
      "...": {},
      "events": {
        "...": ["..."],
        "wasm.ZjZjYzVhYjU4": ["aW8yMTN1MTJp"]
      }
    }
    ```

4. The WebSocket server transmits the transaction event JSONRPC message to Alice. Alice finds the expected attribute key `"wasm.ZjZjYzVhYjU4"` and the notification has been received.


# Concepts

### Notification ID

A globally unique, single-use identifier that is deterministically generated using a cryptographic hash function. Notification IDs can be generated by both contract and client.

An event's Notification ID is used for the custom attribute's key in the transaction log, i.e., `"wasm.${NOTIFICATION_ID}": "${ENCRYPED_NOTIFICATION_DATA}"`.


### Channels

Allows a contract to distinguish between different types of events affecting a recipient. For example, an NFT trading contract might have separate channels for "bids" and "buys".


### Notification Seed

A shared secret between client and contract, used as input key material when deriving Notification IDs. It is required to have high entropy. Therefore, clients are only allowed to modify seeds using digital signatures. See [UpdateSeed](#updateseed-method) for more info.


### Internal Contract Secret

By default, contracts derive a client's Notification Seed using an internal secret not known to ANY party (including admins). This allows clients to obtain their Notification IDs without having to execute a transaction. For an increased privacy guarantee, clients can execute a transaction to set a new Notification Seed.


### Counter

In order to generate a unique Notification ID for each subsequent event, clients and contracts must produce a number only used once (i.e., a nonce) as input to the hash function. They must share an understanding for how/when to increment the nonce so that clients can continue to derive new Notification IDs offline.

To meet these criteria, a simple counter scheme is used to derive new nonce values, meaning that the nonce must increment by exactly one for each new notification. That way, clients and contracts are able to keep their nonce values in sync.


### Notification Data

Contracts may optionally provide supplemental data with each notification. For example, a transfer notification may include the sender's address and token amount, encrypted in the attribute's value, i.e., `"wasm.${NOTIFICATION_ID}": "${ENCRYPED_NOTIFICATION_DATA}"`.

Developers looking to take advantage of this option should understand that ALL notifications (including decoys) will need to pad notification data to some predetermined maximum length in order to avoid privacy leaks. It is generally advised to design such payloads to be as short as possible.



# ListChannels Query

Public query to list all notification channels.

Query:
```json
{
  "list_channels": {},
}
```

Response:
```json
{
  "list_channels": {
    "channels": [
      "<id of channel 1>",
      "...",
      "<id of channel N>"
    ]
  }
}
```

# ChannelInfo Query

Authenticated query allows clients to obtain the seed, counter, and Notification ID of a future event, for a specific channel.

Query:
```json
{
  "channel_info": {
    "channel": "<id of channel>"
  }
}
```

Response:
```json
{
  "channel_info": {
    "channel": "<same as query input>",
    "seed": "<shared secret in base64>",
    "counter": "<current counter value>",
    "next_id": "<the next Notification ID>",
    "as_of_block": "<scopes validity of this response>",
  }
}
```

The response includes the Notification ID of the next event in the given channel affecting the given viewer (who is specified in the authentication data, depending on whether a query permit or viewer key is used).

The response also provides the viewer's current seed for the given channel, allowing the client to derive future Notification IDs for this channel offline (i.e., without having to query the contract again).


# UpdateSeed Method

Allows clients to set a new shared secret. In order to guarantee the provided secret has high entropy, clients must submit a signed document and signature to be verified before the new shared secret is accepted.

The signed document follows the query permit format.

Request:
```json
{
  "update_seed": {
    "channel": "<id of channel>",
    "document": {
      "chain_id": "secret-4",
      "account_number": "0",
      "sequence": "0",
      "msgs": [
        {
          "type": "notification_seed",
          "value": {
            "contract": "<bech32 address of contract>",
            "previous_seed": "<base64-encoded value of previous seed>"
          }
        }
      ],
      "fee": {
        "amount": [
          {
            "denom": "uscrt",
            "amount": "0"
          }
        ],
        "gas": "1"
      },
      "memo": ""
    },
    "signature": "<base64-encoded signature of document>"
  }
}
```

Response (same as [`channel_info` query](#channelinfo-query) response):
```json
{
  "update_seed": {
    "channel": "<same as query input>",
    "seed": "<shared secret in base64>",
    "counter": "<current counter value>",
    "next_id": "<the next Notification ID>",
    "as_of_block": "<scopes validity of this response>",
  }
}
```


## Notification ID Algorithm

Pseudocode for generating Notification IDs:
```
fun notificationIDFor(channelId, recipientAddr) {
  // counter reflects the nth notification for the given recipient in the given channel
  let counter := numNotificationsFor[recipientAddr][channelId]

  // recipient has a shared secret with contract
  let sharedSecret = sharedSecretsTable[recipientAddr]

  // use shared secret for seed
  if exists(sharedSecret):
    seed := sharedSecret
  // otherwise, derive seed using contract's internal secret
  else:
    seed := hkdf(ikm=contractInternalSecret, info=canonical(recipientAddr))

  // compute notification ID for this event
  let material = concat(channelId, ":", counter)
  let notificationID := hmac_sha256(seed, material)

  return notificationID
}
```


## Privacy Considerations

The following section describes a hypothetical side-chain attack. It's important however to note the similarity to attacks based on storage access patterns (i.e., spicy printf), which actually expose more private data and should therefore be considered a greater threat. In both cases, using a system of decoys is the current best mitigation in practice.

If a contract action allows _any sender_ to trigger a notification for some recipient, then there is a risk that an attacker could perform a side-chain attack to precompute a victim's next Notification ID for a specific channel within a contract.

For example, Mallory could fork the chain, transfer 10 token X to Alice, record the emitted Notification ID, and then wait to observe that same Notification ID on the actual chain. At that point, Mallory would be able to deduce that _someone_ transferred _some amount_ of token X to Alice.

Notice that the threat model looks very different if the contract only allowed friends of Alice to transfer tokens to her account (i.e., no longer _any sender_). In that case, only a friend of Alice would be able to perform the attack.


### Decoys

To mitigate privacy risks exposed by side-chain attacks, contracts should employ a system of decoys.

Due to the fact that most vulnerable actions (e.g., transfers) tend to also be those most affected by storage access patterns, contracts should strive to emulate every part of the action with decoys.

For example, when Alice transfers token 10 X to Bob, the contract should also transfer 0 token X to Charlie, and 0 token X to David, causing the full repercussions for both Charlie and David as decoy recipients. Bob, Charlie and David should all receive a new record in their transfer history, even though Charlie and David will each see that 0 tokens were transferred to their account. The purpose of updating their transfer history is to create storage access pattern noise, so that an attacker cannot deduce which recipient's balance actually changed. Similarly, Bob, Charlie and David will all receive a notification; although Charlie and David will know to ignore it when they decrypt the Notification Data and find that 0 tokens were transferred to their account.
