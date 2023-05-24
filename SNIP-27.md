# SNIP-27 - Private Push Notifications

## Motivation 

Wallets and dApps currently resort to a polling-based approach in order to notice changes to a user's private state within a contract.

For example, a dApp might periodically query a set of SNIP-2x contracts in order to discover a new incoming transfer.

However, this approach is rather inefficient and can create unwanted load on query nodes. Additionally, there is no clear best practice for determining an optimal polling rate.


## Proposed Solution

Using `plaintext_log`, compliant contracts will emit Cosmos events that contain globally unique, single-use "Notification Ids", creating a push-based notification service.

Users may then subscribe to events on the Tendermint RPC Websocket using a filter in order to receive notifications from the contract.

Once a notification is observed, the receiver knows to invalidate its local cache of the user's private contract state (and possibly query the contract to update its cache).


## Example

Pseudocode for notifications (using a SNIP-2x transfer as example):
```
fun transferNotificationIdFor(recipientAddr) {
  // to allow for multiple types of events
  let eventType := "transfer"

  // each event type should have a way of producing a value here that will never repeat (e.g., a strictly increasing monotonic function)
  let sequence := len(transferHistory[recipientAddr])

  // at this point, we can conceptualize the event as having an id
  let eventId := eventType || recipientAddr || sequence

  // default to using an internal secret
  let notificationKey := contractInternalSecret

  // recipient has a shared secret with contract; use it instead
  let sharedSecret = sharedSecretsTable[recipientAddr]
  if exists(sharedSecret):
    notificationKey := sharedSecret

  // compute notification id for this event
  let notificationId := sha256(notificationKey || eventId)

  return notificationId
}
```

## Actors

### 1. Anonymous Recipient

Users should be able to receive notifications without ever having to negotiate a shared secret with the contract.

Scenario:
> Without ever having to execute a tx, a user queries the contract to get their own private and personal "next incoming transfer" notification id.
> 
> User subscribes to the RPC websocket stream using the returned notification id to filter events.
> 
> A successful transaction emits the user's "next incoming transfer" notification id. The user queries then contract for the transfer information.
> 
> The user may now query the contract again to obtain their new "next incoming transfer" notification id.


### 2. Registered Recipient

Users should be able to compute their own notification ids offline, rather than having to query the contract for each and every new notification id.

Scenario:
> A user signs a SignDoc offline and then submits it to the contract.
>
> The contract stores the signature in a lookup table keyed by the sender's address. The signature is now used as a shared secret. The SignDoc signature is also verified to ensure the shared secret has high entropy.
> 
> The recipient can now compute all of its notification ids for the given contract offline.


### 3. Notification Initiator

Contracts only emit events when a transaction causes a state change that affects a third party (e.g., a transfer affecting a recipient's balance).

Scenario: 
> A sender executes a transfer. The contract computes the notification id for the recipient and emits it.


## Known Privacy Risks and Mitigations

### Untrusted Query Node
A potential privacy risk would involve an untrusted query node if the only event types emitted by the contract are directed and relational (e.g., it only emits transfer events where the sender of a tx causes the recipient's notification id to be emitted).

#### Threat
A malicious query node serving the websocket could log a recipient's notification id (supplied in their JSONRPC request filter) with the eventual sender of a transaction by matching the event, thus correlating the sender/recipient pair of a transfer.

#### Mitigation
A system could be implemented such that registered recipients negotiate a decoy notification id generation scheme with the contract. The contract would then emit a finite set of decoy notification ids with every tx, and the user would also subscribe to its own decoy notification ids. Then, only the contract and the recipient would know which notification ids are authentic.
