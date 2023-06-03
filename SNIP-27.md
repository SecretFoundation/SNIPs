# SNIP-52 - Private Push Notifications

Allows clients to receive notifications when certain events within a smart contract affect them. For example: token transfers, direct messaging, turn-based gaming, announcements, and so on.

* [Introduction](#introduction)
  * [Motivation](#motiviation)
  * [Overview](#overview)
    * [How it works](#how-it-works-high-level)
    * [Example flow](#example-flow)
* [Concepts]

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

## Notification ID

A globally unique, single-use identifier that is deterministically generated using a cryptographic hash function. Notification IDs can be generated by both contract and client.

An event's Notification ID is used for the custom attribute's key in the transaction log, i.e., `"wasm.${NOTIFICATION_ID}": "${ENCRYPED_NOTIFICATION_DATA}"`.


## Channels

Allows a contract to distinguish between different types of events affecting a recipient. For example, an NFT trading contract might have separate channels for "bids" and "buys".


## Notification Seed

A shared secret between client and contract, used as input key material when deriving Notification IDs.


## Internal Contract Secret

By default, contracts derive a client's Notification Seed using an internal secret not known to ANY party (including admins). This allows clients to obtain their Notification IDs without having to execute a transaction. For an increased privacy guarantee, clients can execute a transaction to set a new Notification Seed.


## Notification Data

Contracts may optionally provide supplemental data with each notification. For example, a transfer notification may include the sender's address and token amount, encrypted in the attribute's value, i.e., `"wasm.${NOTIFICATION_ID}": "${ENCRYPED_NOTIFICATION_DATA}"`.

Developers looking to take advantage of this option should understand that ALL notifications (including decoys) will need to pad notification data to some predetermined maximum length in order to avoid privacy leaks. It is generally advised to design such payloads to be as short as possible.


## Decoy Notifications

In order to maintain privacy for all users, contracts should always emit the same number of attributes in the transaction log, even if the execution does not warrant an actual notification. A "Decoy Notification" simply refers to the practice of generating fake Notification IDs to add to the transaction log.



# NextNotificationId Query

Allows clients to obtain the Notification ID of a future event, for a specific channel.

```ts
type NextNotificationIdQuery = OptionallyWithPermit<{
  next_notification_id: {
    channel: string;  // e.g., "transfers", "direct-messages", "turns"
    viewer_info?: {
      address: string;
      viewing_key: string;
    };
  };
}>

interface NextNotificationIdResponse {
  next_notification_id: {
    channel: string;  // same as the query's input
    seed: Base64;  // shared secret unique to this channel
    next_id: string;  // the next Notification ID
    as_of_block: Uint128;  // scopes validity of this response data
  };
}
```

The response includes the Notification ID of the next event in the given channel affecting the given viewer (who is specified in the authentication data, depending on whether a query permit or viewer key is used).

The response also provides the viewer's current seed for the given channel, allowing the client to derive future Notification IDs for this channel offline (i.e., without having to query the contract again).




## Example

Pseudocode for notifications (using a SNIP-2x transfer as example):
```
fun transferNotificationIDFor(recipientAddr) {
  // to allow for multiple types of events
  let eventType := "transfer"

  // each event type should have a way of producing a value here that will never repeat (e.g., a strictly increasing monotonic function)
  let sequence := len(transferHistory[recipientAddr])

  // at this point, we can conceptualize the event as having an ID
  let eventID := eventType || recipientAddr || sequence

  // default to using an internal secret
  let notificationKey := contractInternalSecret

  // recipient has a shared secret with contract; use it instead
  let sharedSecret = sharedSecretsTable[recipientAddr]
  if exists(sharedSecret):
    notificationKey := sharedSecret

  // compute notification ID for this event
  let notificationID := sha256(notificationKey || eventID)

  return notificationID
}
```

## Actors

### 1. Anonymous Recipient

Users should be able to receive notifications without ever having to negotiate a shared secret with the contract.

Scenario:
> Without ever having to execute a tx, a user queries the contract to get their own private and personal "next incoming transfer" notification ID.
> 
> User subscribes to the RPC websocket stream using the returned notification ID to filter events.
> 
> A successful transaction emits the user's "next incoming transfer" notification ID. The user queries then contract for the transfer information.
> 
> The user may now query the contract again to obtain their new "next incoming transfer" notification ID.


### 2. Registered Recipient

Users should be able to compute their own notification IDs offline, rather than having to query the contract for each and every new notification ID.

Scenario:
> A user signs a SignDoc offline and then submits it to the contract.
>
> The contract stores the signature in a lookup table keyed by the sender's address. The signature is now used as a shared secret. The SignDoc signature is also verified to ensure the shared secret has high entropy.
> 
> The recipient can now compute all of its notification IDs for the given contract offline.


### 3. Notification Initiator

Contracts only emit events when a transaction causes a state change that affects a third party (e.g., a transfer affecting a recipient's balance).

Scenario: 
> A sender executes a transfer. The contract computes the notification ID for the recipient and emits it.


## Known Privacy Risks and Mitigations

### Untrusted Query Node
A potential privacy risk would involve an untrusted query node if the only event types emitted by the contract are directed and relational (e.g., it only emits transfer events where the sender of a tx causes the recipient's notification ID to be emitted).

#### Threat
A malicious query node serving the websocket could log a recipient's notification ID (supplied in their JSONRPC request filter) with the eventual sender of a transaction by matching the event, thus correlating the sender/recipient pair of a transfer.

#### Mitigation
A system could be implemented such that registered recipients negotiate a decoy notification ID generation scheme with the contract. The contract would then emit a finite set of decoy notification IDs with every tx, and the user would also subscribe to its own decoy notification IDs. Then, only the contract and the recipient would know which notification IDs are authentic.
