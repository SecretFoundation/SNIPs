# SNIP-52 - Private Push Notifications

Allows clients to receive notifications when certain events within a smart contract affect them. For example: token transfers, direct messaging, turn-based gaming, and so on.

```
Revision: #2
Revised on: 2024-03-12
```

- [SNIP-52 - Private Push Notifications](#snip-52---private-push-notifications)
- [Introduction](#introduction)
  - [Motivation](#motivation)
  - [Overview](#overview)
    - [How it works (high-level)](#how-it-works-high-level)
    - [Example flow](#example-flow)
    - [Diagram](#diagram)
    - [Remarks on the above example](#remarks-on-the-above-example)
- [Channel Modes](#channel-modes)
    - [Single Recipient: Counter Mode & TxHash Mode](#notifying-a-single-recipient-counter-mode--txhash-mode)
    - [Multiple Recipients: Bloom Mode](#notifying-multiple-recipients-bloom-mode)
      - [Bloom filter](#bloom-filter)
      - [Arbitrary data](#attaching-private-data)
- [Concepts](#concepts)
    - [Notification ID](#notification-id)
    - [Channels](#channels)
    - [Notification Seed](#notification-seed)
    - [Internal Contract Secret](#internal-contract-secret)
    - [Counter](#counter)
    - [Notification Data](#notification-data)
      - [CBOR and CDDL](#cbor-and-cddl)
- [Queries and Methods](#queries-and-methods)
  - [ListChannels Query](#listchannels-query)
  - [ChannelInfo Query](#channelinfo-query)
  - [WithPermit Query](#withpermit-query)
      - [QueryWithPermit](#querywithpermit)
        - [WithPermit `query` Parameter](#withpermit-query-parameter)
  - [UpdateSeed Method](#updateseed-method)
- [Algorithms](#algorithms)
  - [Contract Internal Secret Derivation](#contract-internal-secret-derivation)
  - [Notification Seed Algorithm](#notification-seed-algorithm)
  - [Updating Seed Algorithm](#updating-seed-algorithm)
  - [Notification ID Algorithm](#notification-id-algorithm)
  - [Notification Data Algorithms](#notification-data-algorithms)
  - [Dispatch Notification Algorithm](#dispatch-notification-algorithm)
  - [Cryptography](#cryptography)
    - [HKDF](#hkdf)
    - [Secp256k1](#secp256k1)
    - [HMAC-SHA256](#hmac-sha256)
    - [ChaCha20-Poly1305](#chacha20-poly1305)
  - [Privacy Considerations](#privacy-considerations)
    - [Padding: Constant-Length Notification Data](#padding-constant-length-notification-data)
    - [Decoy Notifications: Constant Log Size](#decoy-notifications-constant-log-size)
    - [Hypothetical Attack in Counter Mode](#hypothetical-attack-in-counter-mode)

# Introduction

## Motivation 

Wallets and dApps currently resort to a polling-based approach in order to notice changes to a user's private state within a contract.

For example, a dApp might periodically query a set of SNIP-2x token contracts to discover a new incoming transfer.

However, this approach of querying contracts every so often is inefficient and can create unwanted load on query nodes. Additionally, there is no clear best practice for determining an optimal polling rate.


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

> NOTE: Example uses fake base64 data for brevity

1. Alice queries the token X contract to get the unique Notification ID for her next incoming transfer:
    ```json
    {
      "channel_info": {
        "channels": ["transfers"]
      }
    }
    ```

2. The contract responds:
    ```json
    {
      "channel_info": {
        "as_of_block": "1131420",
        "channels": [
          {
            "channel": "transfers",
            "mode": "counter",
            "seed": "ecc7f60418aa",
            "counter": "3",
            "next_id": "ZjZjYzVhYjU4",
            "cddl": "transfers=[amount:biguint,sender:bstr]"
          }
        ]
      }
    }
    ```

    Alice now has a globally unique Notification ID the contract will use next time someone transfers tokens to her account.

    Furthermore, Alice now has a seed she can use to derive future Notification IDs offline for subsequent transfer events (i.e., without having to query this contract again)

3. Alice subscribes to execution events on the token contract:
    ```json
    {
      "jsonrpc": "2.0",
      "id": "0",
      "method": "subscribe",
      "params": {
        "query": "wasm.contract_address='secret1ku936rhzqynw6w2gqp4e5hsdtnvwajj6r7mc5z'"
      }
    }
    ```

    Alice will now receive a JSONRPC message for each new execution of the contract, from which she can search for her unique Notification ID.
    
    If operating in counter mode, and Alice trusts that the WebSocket server won't record her activity, she can optionally use a filter in the `query` field of her subscription message, e.g., `wasm.snip52:ZjZjYzVhYjU4 EXISTS` .

4. Some time later, Bob executes a SNIP-20 transfer of token X to Alice's account:
    ```json
    {
      "transfer": {
        "recipient": "Alice",
        "amount": "100",
      }
    }
    ```

5. The contract derives the next transfer Notification ID for the recipient (Alice) and adds it as a custom attribute to the transaction log.
    ```json
    {
      "...": {},
      "events": {
        "...": ["..."],
        "wasm.snip52:ZjZjYzVhYjU4": ["aW8yMTN1MTJp"]
      }
    }
    ```

6. The WebSocket server transmits the transaction event JSONRPC message to Alice. Alice finds the expected attribute key `"wasm.snip52:ZjZjYzVhYjU4"` and the notification has been received.


### Diagram

![SNIP-52 Notification Map](./media/notification-map.png)


### Remarks on the above example

Once Alice has obtained her Notification Seed for the desired channel, she no longer needs to query the contract and steps 4-6 can repeat ad infinitum.


# Channel Modes

Each channel can operate in one of several modes: Counter Mode, TxHash Mode, or Bloom Mode. A channel's mode should either be hardcoded or set during contract initialization. It should never change.

> NOTE: Additional channel modes are reserved for future revisions. Implementations should not assume that only the given modes exist. Therefore, implementations should NOT use a _default_ case or _else_ branch to select the remaining condition.


## Single recipient: Counter Mode & TxHash Mode

Counter Mode and TxHash Mode are for channels that deliver each notification to a single recipient, for example, a direct token transfer, direct messaging, or a two-player game.

There is a trade-off between these two modes. Counter Mode is easier on clients but less secure.

In Counter Mode:
 - ✅ clients only have to recompute a channel's notification ID each time they receive a notification from that channel
 - ✅ clients are able to use node APIs such as `tx_search` to search back through history and find a missed notification
 - ✅ clients are able to query the contract to obtain their next notification ID, allowing them to bypass much of the SNIP-52 client-side implementation
 - ❌ an attacker could, in theory, de-anonymize notification IDs using a sophisticated [side-chain attack](#hypothetical-attack-in-counter-mode)

In TxHash Mode:
 - ✅ notifications are immune to side-chain attacks
 - ❌ clients must recompute their notification ID for every execution tx witnessed on the given contract

In summary, TxHash Mode is more secure than Counter Mode, but comes at the cost of more work for the client (although the processing heft is likely neglible). On the other hand, with Counter Mode, clients are able to easily search tx history for missed notifications, and clients can bypass computing Notification IDs altogether by querying the contract.


## Multiple recipients: Bloom Mode

Bloom Mode allows contracts to deliver notifications to multiple users at once, for example, batch transfers, group messaging, or in a multi-player game.

Unlike single recipient channels, the attribute key for Bloom mode chanels is constant _across events_. For example, a channel with the ID `group_message` will always output to the attribute key `"wasm.snip52:#group_message"`.

The attribute value is then made up of two major parts, the bloom filter bytes followed by the notification data bytes: `${BLOOM_FILTER}${NOTIFICATION_DATA}`. Both parts are constant length, and the notification data is optional (i.e., it may have length 0). If you _would_ like to deliver notification data to each of the recipients while maintaining privacy, see [more here](#attaching-private-data).

### Bloom filter

The [bloom filter](https://en.wikipedia.org/wiki/Bloom_filter) is a data structure that allows for set membership to be encoded into a small space. Since false positive matches are possible (but ideally rare, depending on the parameters), it is only used by clients as a first step to determining whether a given notification affects them or not. In other words, if a client gets a hit on the bloom filter then they know to check the notification data and/or query the contract state to test whether they actually received a notification.

Contract developers need to choose parameters _m_, _k_ and _h_ for each channel's bloom filter, where _m_ MUST be divisible by 8. Larger values of _m_ are needed for larger group sizes, but consume more space in the logs, whereas _k_ should be chosen to be optimal over the range of the expected number of recipients. Finally, _h_ is the hash function from which _k_ adjacent chunks, each of length $\log_{2}(m)$ bits, will be extracted.

This tool can assist with picking _m_ and _k_ parameters depending on the use case: https://hur.st/bloomfilter/?n=16&p=&m=512&k=15 . 

Choosing hash function _h_ should be (a) cryptographically secure to prevent preimage attacks from revealing recipients' notification IDs, (b) uniformly random to ensure that the bloom filter is effective for all potential clients listening to the notification feed, and (c) produce at least $k * \log_{2}(m)$ bits.

The bloom filter MUST be set by extracting _k_ adjacent values, each $\log_{2}(m)$ bits in size, from the hash digest starting at the leftmost bit. Each value MUST be read as an unsigned int in big-endian, which is then used to set the _n_-th bit in the bloom filter, where index 0 corresponds to the rightmost bit.

For example, assuming a filter using params _m_ = 512 and _k_ = 15, we can select the cryptographic hash function _h_ = sha256, since $k * \log_{2}(m) = 15 * \log_{2}(512) = 15 * 9 = 135 \leq 256$. The 15 hash values `[k_0..k_14]` would be derived as follows:
```
let bloomFilter := bytes(64)

for recipientAddr in recipients:
   let notificationId := notificationIDFor(recipientAddr, channelId, {txHash})
   let bloomHash := sha256(notificationId)

   for i in 0..15:
      let k_i := sliceBits(bloomHash, i*9, (i+1)*9)
  
      let toggle := uintToBytes(1 << bitsToUintBe(k_i))
      bloomFilter := orBytes(bloomFilter, toggle)
```

### Attaching private data

Contract developers are free to choose a data scheme to complement their bloom filter, however contracts SHOULD design and return a machine-readable representation of that schema in responses to the [ChannelInfo Query](#channelinfo-query).


The format for the data scheme is made up of datatypes that resemble those from [EVM's textual ABI](https://docs.soliditylang.org/en/develop/abi-spec.html#types), namely `bool`, `uint<M>`, `address`, `bytes<M>`, and `<type>[M]`, with the additional custom types `struct`, and the special `packet[M]`.


All `uint<M>` datatypes SHOULD be assumed to be clamped. If the client observes the max value for a `uint<M>`, then the contract ran out of bits and the client should query the contract for the actual value.

The following typings formally describe it in TypeScript:
```ts
type Uint = `${bigint}`;  // 0, 1, 2, ...
type UintSizes = '8' | '16' | '24' | '32' | '40' | '48' /* ... */ | '248' | '256';
type List<Datatype> = `${Datatype}[${Uint}]`;

type Primitive = `uint${UintSizes}` | 'address' | `bytes${Uint}`;

type FlatDescriptor = {
  type: Primitive | List<Primitive> | List<List<Primitive>>;
  label: string;
  description?: string;
};

type StructDescriptor = {
  type: 'struct' | List<'struct'>;
  members: Descriptor[];
  label: string;
  description?: string;
};

type DataDescriptor = FlatDescriptor | StructDescriptor;

export type Descriptor = DataDescriptor | {
  type: `packet[${Uint}]`;
  version: number;
  packetSize: number;
  data: DataDescriptor;
};
```

##### Example schema:

```json
{
  "type": "packet[16]",
  "version": "1",
  "packetSize": 108,
  "data": {
    "type": "struct",
    "members": [
      {
        "type": "uint64",
        "label": "amount"
      },
      {
        "type": "address",
        "label": "sender"
      },
      {
        "type": "bytes80",
        "label": "memo",
        "description": "UTF8-encoded memo string"
      }
    ]
  }
}
```

#### Packet datatype

The `packet[M]` datatype is a special case that makes it convenient to encrypt different notification data for each recipient (i.e., only the intended recipient will be able to decrypt a packet's contents), up to a maximum of `M` recipients.

Each packet must be fixed width so that clients can search for their packet at predictable offsets (also, a variable-width encoding scheme would leak data to other recipients) so CBOR encoding does not make sense here. This width is specified in bytes as the `packetSize`.

The `version` specifies the revision of this datatype, and at the time of this writing should be set to `1`. Clients should assert that this version matches in their implementation.

#### Packet specification version 1

Each packet is composed of two major parts: `${PACKET_ID}${CIPHERTEXT}` .

The packet ID is simply the 8 leftmost bytes of the notification ID for the intended recipient in the given channel.

Clients that found a hit on the bloom filter will then scan the packets searching for a packet ID matching the 8 leftmost bytes of their notification ID. Each next packet ID will start at exactly `packetSize` bytes after the end of the previous one.

The ciphertext is `packetSize` bytes long, the same length as its plaintext equivalent. When encrypting/decrypting packet data, the remaining 24 bytes of the notification ID are used as the key material, referred to here as the packet IKM. In other words, the packet IKM is the 24 bytes _immediately following_ the 8 leftmost bytes of the notification ID.

Encrypting and decrypting packet data is performed using a one-time pad by XOR'ing the plaintext/ciphertext with the packet key. The packet key is derived using one of two techniques, depending on `packetSize`:
 1. If `packetSize` is less than or equal to 24 bytes, the packet key is simply the leftmost bytes of the packet IKM, up to `packetSize` bytes.
 2. If `packetSize` is greater than or equal to 25 bytes, then HKDF is performed on the 192-bit packet IKM with 256 bits of zero-valued salt, empty info and the SHA512 hash function, deriving exactly `packetSize * 8` bits. Those derived bits become the packet key.

```
let packetSize := length(packetPlaintext)
let packetId := slice(notificationId, 0, 8)
let packetIkm := slice(notificationId, 8, 32)
let packetKey

if packetSize <= 24:
  packetKey := slice(packetIkm, 0, packetSize)
else:
  packetKey := hkdfSha512(ikm=packetIkm, salt=bytes(64), info="", length=packetSize*8)

let packetCiphertext := xorBytes(packetPlaintext, packetKey)
let packetBytes = concat(packetId, packetCiphertext)
```


# Concepts

### Event Log Attributes

Notifications work by adding attributes to the Tendermint Event log. Since these originate from the contract, they are always prefixed by `"wasm."`. Additionally, all SNIP-52 notifications MUST insert the `"snip52:"` attribute key prefix, which helps identify SNIP-52 events in downstream services and on the client.

An event attribute can take on one of the following forms:
 - `"wasm.snip52:${NOTIFICATION_ID}": "${ENCRYPTED_NOTIFICATION_DATA}"` for single-recipient notifications operating in [Counter Mode or TxHash Mode](#notifying-a-single-recipient-counter-mode--txhash-mode).
 - `"wasm.snip52:#${CHANNEL_ID}": "${BLOOM_FILTER}${ARBITRARY_NOTIFICATION_DATA}"` for multi-recipient notifications operating in [Bloom Mode](#notifying-multiple-recipients-bloom-mode).


### Notification ID

A globally unique, single-use identifier that is deterministically generated using a cryptographic hash function. Notification IDs can be generated by both contract and client.

An event's Notification ID is used for the custom attribute's key in the transaction log when the channel notifies a single recipient, i.e., `"wasm.snip52:${NOTIFICATION_ID}"`, and is used to compute bloom filter hashes when the channel notifies multiple recipients.


### Channels

Allows a contract to distinguish between different types of events affecting a recipient. For example, an NFT trading contract might have separate channels for "bids" and "buys".


### Notification Seed

A shared secret between client and contract, used as input key material when deriving Notification IDs. It is required to have high entropy. Therefore, clients are only allowed to modify seeds using digital signatures. See [UpdateSeed](#updateseed-method) for more info.


### Internal Contract Secret

By default, contracts derive a client's Notification Seed using an internal secret not known to ANY party (including admins). This allows clients to obtain their Notification IDs without having to execute a transaction. For an increased privacy guarantee, clients can execute a transaction to set a new Notification Seed.


### Counter

In order to generate a unique Notification ID for each subsequent event, clients and contracts MUST produce a number only used once (i.e., a nonce) as input to the hash function. They must share an understanding for how/when to increment the nonce so that clients can continue to derive new Notification IDs offline.

For channels operating in [Counter Mode](#modes-counter-mode-vs-txhash-mode), a simple counter scheme is used to derive new nonce values, meaning that the nonce MUST increment by exactly one for each new notification. That way, clients and contracts are able to keep their nonce values in sync.


### Notification Data

Contracts MAY optionally provide supplemental data with each notification. For example, a transfer notification may include the sender's address and token amount, encrypted in the attribute's value, i.e., `"wasm.snip52:${NOTIFICATION_ID}": "${ENCRYPED_NOTIFICATION_DATA}"`.

Developers looking to take advantage of this option should understand that ALL notifications (including decoys) will need to pad notification data to some predetermined maximum length in order to avoid privacy leaks. It is generally advised to design such payloads to be as short as possible.


#### CBOR and CDDL

Contracts SHOULD encode notification data using [CBOR](https://cbor.io/spec.html), where the top-level element is always a tuple. Furthermore, contracts SHOULD provide a Concise Data Definition Language (CDDL) definition string in the [ChannelInfo Query](#channelinfo-query) response (under the `"cddl"` key) which describes the payload.

For maximum interoperability:
1. Top-level CBOR value should be a tuple
2. CDDL type definition name should match its channel ID
3. All elements should be annotated with human-readable names in the CDDL

Walking through the basic SNIP-2x "transfers" example, a notification would want to include the amount received and the sender. Since the token amount could possibly exceed the range of `uint64`, we use `biguint` instead. To keep the payload as short as possible, we transmit the sender's address in canonical byte form. An appropriate CDDL for its channel would look like this:
```cddl
transfers = [
  amount: biguint,  ; number of indivisible token units
  sender: bstr,     ; byte sequence of sender's canonical address
]
```

Evaluating this against the rubric above:
1. The top-level value is a CBOR tuple ✔
2. The CDDL type definition "transfers" matches the channel ID ✔
3. All elements in the tuple ("amount" and "sender") are named for human-readability ✔


Now imagine Bob transfers 1.25 token X to Alice. The corresponding information would be as follows:
```
amount: 1250000 micro TKN
sender: secret1dg4gt6fc2avp2ywgvrxxmptc670av0372u2gv5
```

Encoding this information in CBOR according the above "transfers" schema results in the following payload (shown here in diagnostic notation):
```cbor-diag
[1250000, h'6a2a85e93857581511c860cc6d8578d79fd63e3e']
```

For more information about CBOR and CDDL:
 - [Specs](https://cbor.io/spec.html)
   - [CBOR Specification](https://www.rfc-editor.org/rfc/rfc8746.html)
   - [CDDL Specification](https://cbor-wg.github.io/cddl/draft-ietf-cbor-cddl.html)
 - [Tools](https://cbor.io/tools.html)
 - [Impls](https://cbor.io/impls.html)
 - [CDDL Examples](https://github.com/cbor-wg/cddl/blob/master/cddl-examples.md)


# Queries and Methods


## ListChannels Query

Public query to list all notification channels.

Query:
```json
{
  "list_channels": {},
}
```

<details>
  <summary>Show TypeScript equivalent</summary>

  ```ts
  type ListChannelsQueryMsg = {
    list_channels: {};
  };
  ```
</details>


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

<details>
  <summary>Show TypeScript equivalent</summary>

  ```ts
  type ListChannelsQueryResponse = {
    list_channels: {
      channels: string[];
    };
  };
  ```
</details>


## ChannelInfo Query

Authenticated query allows clients to obtain the seed, counter, and Notification ID of a future event, for a specific set of channels.

Query:
```json
{
  "channel_info": {
    "channels": ["<id of channel>", "<...optional list of additional channel ids>"],
    "txhash": "<optional 64 hex characters of a transaction hash>",
    "viewer": {
      "address": "<address of the querier>",
      "viewing_key": "<viewer's key>"
    }
  }
}
```

<details>
  <summary>Show TypeScript equivalent</summary>

  ```ts
  type ChannelInfoQueryMsg = {
    channels: string[];
    txhash?: string;
    viewer: {
      address: string;  // bech32
      viewing_key: string;
    };
  };
  ```
</details>

| Name     | Type                                 | Description                                       | Optional |
|----------|--------------------------------------|---------------------------------------------------|----------|
| channels | array of strings                     | A list of channel IDs                             | no       |
| txhash   | string                               | A transaction hash to compute the Notification ID | yes      |
| viewer   | [ViewerInfo](SNIP-721.md#viewerinfo) | The address and viewing key performing this query | no       |


Response:
> NOTE: The shape of each item in the `channels` array depends on its `mode` value (either `"txhash"`, `"counter"` or `"bloom"`). See below for more details.
```json
{
  "channel_info": {
    "as_of_block": "<scopes validity of this response>",
    "channels": [
      {
        "channel": "<channel id, corresponds to query input>",
        "mode": "txhash",
        "seed": "<shared secret in base64>",
        "cddl": "<optional CDDL schema definition string for the CBOR-encoded notification data>",
        "answer_id": "<if txhash argument was given, this will be its computed Notification ID>"
      },
      {
        "channel": "<channel id, corresponds to query input>",
        "mode": "counter",
        "seed": "<shared secret in base64>",
        "cddl": "<optional CDDL schema definition string for the CBOR-encoded notification data>",
        "counter": "<current counter value>",
        "next_id": "<the next Notification ID>"
      },
      { "...": "..." }
    ]
  }
}
```

<details>
  <summary>Show TypeScript equivalent</summary>

  ```ts
  type ChannelInfoQueryResponse = {
    as_of_block: string;  // uint64
    channels: ChannelInfo[];
  };

  type ChannelInfo = {
    channel: string;
    seed: string;  // base64
    cddl?: string;  // cddl
  } & ({
    mode: "counter";
    counter: string;  // uint64
    next_id: string;
  } | {
    mode: "txhash";
    answer_id?: string;
  } | {
    mode: "bloom";
    parameters: {
      m: number;
      k: number;
    };
    data: CwAbiLikeDescriptor;
    answer_id?: string;
  });
  ```
</details>


If a channel is operating in Counter Mode, given by `"mode": "counter"`, then its response row includes the current counter value (under the `counter` key) and the next Notification ID (under the `next_id` key) corresponding to the given channel affecting the current viewer (who was specified in the query authentication data, depending on whether a query permit or ViewerInfo was used).

The response also provides the viewer's current seed for each given channel, allowing the client to derive future Notification IDs for this channel offline (i.e., without having to query the contract again).


## WithPermit Query

SNIP-52 contracts may optionally implement query permits as specified in [SNIP-24](SNIP-24.md).

WithPermit wraps permit queries in the [same manner](SNIP-24.md#WithPermit) as SNIP-24.

Query:
```json
{
  "with_permit": {
    "permit": {
      "params": {
        "permit_name": "some_name",
        "allowed_tokens": ["addr_1", "addr_2", "..."],
        "chain_id": "some_chain_id",
        "permissions": ["owner"]
      },
      "signature": {
        "pub_key": {
          "type": "tendermint/PubKeySecp256k1",
          "value": "33_bytes_of_secp256k1_pubkey_as_base64"
        },
        "signature": "64_bytes_of_secp256k1_signature_as_base64"
      }
    },
    "query": {
      "QueryWithPermit_variant_defined_below": { "...": "..." }
    }
  }
}
```

<details>
  <summary>Show TypeScript equivalent</summary>

  ```ts
  type WithPermitQuery = {
    permit: {
      params: {
        permit_name: string;
        allowed_tokens: string[];  // bech32s
        chain_id: string;
        permissions: string[];
      };
      signature: {
        type: "tendermint/PubKeySecp256k1";
        value: string;  // base64
      };
    };
    query: Omit<AuthenticatedQueries, "viewer">;
  };

  type AuthenticatedQueries = ChannelInfoQueryMsg;
  ```
</details>

| Name   | Type                                            | Description                                   | Optional |
|--------|-------------------------------------------------|-----------------------------------------------|----------|
| permit | [Permit](SNIP-24.md#WithPermit)                 | A permit following SNIP-24 standard           | no       |
| query  | [QueryWithPermit (see below)](#QueryWithPermit) | The query to perform and its input parameters | no       |


#### QueryWithPermit
QueryWithPermit is an enum whose single variant correlates with the SNIP-52 query that requires authentication ([ChannelInfo](#channelinfo-query)). The input parameters are the same as the corresponding query other than the absence of [ViewerInfo](#viewerinfo) because the permit supplied with the `WithPermit` query provides both the address and authentication.

* ChannelInfo ([corresponding query](#channelinfo-query))
##### WithPermit `query` Parameter
```json
{
  "query": {
    "channel_info": {
      "channels": ["<id of channel>", "<...optional list of additional channel ids>"],
    }
  }
}
```


## UpdateSeed Method

Allows clients to set a new shared secret. In order to guarantee the provided secret has high entropy, clients must submit a signed document params and signature to be verified before the new shared secret (a SHA-256 hash of the signature) is accepted.

See the [Updating Seed Algorithm](#updating-seed-algorithm) for details on how the contract should handle this message.

The signed document follows the same format as query permits, but with `type` `"notification_seed"` and `value` containing the two fields `contract` and `previous_seed`, both of which the contract will auto-populate when verifying the permit:
```json
{
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
}
```

Request:
```json
{
  "update_seed": {
    "signed_doc": {
      "params": {
        "chain_id": "secret-4",
      },
      "signature": {
        "pub_key": {
          "type": "tendermint/PubKeySecp256k1",
          "value": "<33 bytes of secp256k1 pubkey as base64>"
        },
        "signature": "<64 bytes of secp256k1 signature as base64>"
      }
    }
  }
}
```

<details>
  <summary>Show TypeScript equivalent</summary>

  ```ts
  type UpdateSeedExecMsg = {
    update_seed: {
      signed_doc: {
        params: {
          chain_id: string;
        };
      };
      signature: {
        pub_key: {
          type: "tendermint/PubKeySecp256k1";
          value: string;  // base64
        };
        signature: string;  // base64
      };
    };
  };
  ```
</details>


Response:
```json
{
  "update_seed": {
    "seed": "<shared secret in base64>"
  }
}
```

<details>
  <summary>Show TypeScript equivalent</summary>

  ```ts
  type UpdateSeedExecResponse = {
    update_seed: {
      seed: string;  // base64
    };
  };
  ```
</details>


# Algorithms

## Contract Internal Secret Derivation

Contracts should strive to create an internal secret such that even the creator cannot predict or extract its contents.

Typically, such secrets are generated upon initialization. A suitably strong and robust method for generating this secret combines user-provided entropy and Secret Network's verifiable randomness API. Pseudocode for reference only:
```
fun initializeContract(msg, env) {
  // gather entropy from sender
  let userEntropy := msg.entropy

  // extend entropy with environmental information
  let entropy := concat(
    env.blockHeight,
    env.blockTime,
    env.senderAddress,
    userEntropy
  )

  // the crux: obtain a unique, cryptographically-strong random value associated with this execution
  let seed := env.random()

  // also very important: derive the contract's internal secret using HKDF
  let internalSecret := hkdfSha256(ikm=seed, salt=sha256(entropy), info="contract_internal_secret", length=32)

  // save to storage
  saveInternalSecretToStorage(internalSecret);

  // ...
}
```

> NOTE: The above approach would still allow a malicious contract deployer to hypothetically deduce the contract's internal secret. In order to achieve greater levels of trust, the process of deriving the internal secret would require several subsequent executions where multiple third parties provide additional (secret) entropy. However, such an approach is complex and outside the scope of this document.


## Notification Seed Algorithm

Pseudocode for settling on a seed to use (contract):
```
fun getSeedFor(recipientAddr) {
  // recipient has a shared secret with contract
  let seed := sharedSecretsTable[recipientAddr]

  // no explicit shared secret; derive seed using contract's internal secret
  if NOT exists(seed):
    seed := hkdfSha256(ikm=contractInternalSecret, info=canonical(recipientAddr))

  return seed
}
```


## Updating Seed Algorithm

Pseudocode for verifying [`update_seed`](#updateseed-method) arguments and storing new seed (contract):
```
fun updateSeed(recipientAddr, signedDoc, env) {
  // check that the params are accurate
  assert(signedDoc.params.contract == env.contractAddr)
  assert(signedDoc.params.previous_seed == sharedSecretsTable[recipientAddr])

  // verify that the signature belongs to the sender and is for the given signed document
  verifySecp256k1Signature(env.senderPubKey, signedDoc.signature, {
    "chain_id": signedDoc.params.chain_id,
    "account_number": "0",
    "sequence": "0",
    "msgs": [
      {
        "type": "notification_seed",
        "value": {
          "contract": signedDoc.params.contract,
          "previous_seed": signedDoc.params.previous_seed
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
  })

  // hash the signature to get the 32 byte shared secret
  let sharedSecret := sha256(signedDoc.signature.signature)

  // save the shared secret to storage associated with the given recipient
  saveToSharedSecretsTable(recipientAddr, sharedSecret)
}
```


## Notification ID Algorithm

Pseudocode for generating Notification IDs (both contract & client):
```
fun notificationIDFor(contractOrRecipientAddr, channelId, env) {
  let salt := nil

  // depending on which mode the channel operates in
  if inCounterMode(channelId):
    // counter reflects the nth notification for the given contract/recipient in the given channel
    let counter := getCounterFor(contractOrRecipientAddr, channelId)
    salt := uintToDecimalString(counter)

  // otherwise, channel is in TxHash Mode
  else:
    salt := env.txHash

  // compute notification ID for this event
  let seed := getSeedFor(contractOrRecipientAddr)
  let material := concatStrings(channelId, ":", salt)
  let notificationID := hmac_sha256(key=seed, message=utf8ToBytes(material))

  return notificationID
}
```


## Notification Data Algorithms

Contracts are encouraged to use [CBOR](https://cbor.io/) to encode/decode information in the notification data.

Pseudocode for encrypting data into notifications (contract):
```
fun encryptNotificationData(recipientAddr, channelId, plaintext, env) {
  // ChaCha20 expects a 96-bit (12 bytes) nonce. we will combine two 12 byte buffers to create nonce
  let saltBytes := nil

  // depending on which mode the channel operates in
  if inCounterMode(channelId):
    // counter reflects the nth notification for the given recipient in the given channel
    let counter := getCounterFor(recipientAddr, channelId)

    // encode uint64 counter in BE and left-pad with 4 bytes of 0x00 to make 12 bytes
    saltBytes := concat(zeros(4), uint64BigEndian(counter))

  // otherwise, channel is in TxHash Mode
  else:
    // take first 12 bytes of tx hash (make sure to decode the hex string)
    saltBytes := slice(hexToBytes(env.txHash), 0, 12)

  // take the first 12 bytes of the channel id's sha256 hash
  let channelIdBytes := slice(sha256(utf8ToBytes(channelId)), 0, 12)

  // produce the nonce by XOR'ing the two previous 12-byte results
  let nonce := xorBytes(channelIdBytes, saltBytes)

  // right-pad the plaintext with 0x00 bytes until it is of the desired length (keep in mind, payload adds 16 bytes for tag)
  let message := concat(plaintext, zeros(DATA_LEN - len(plaintext)))

  // construct the additional authenticated data
  let aad := concatStrings(env.blockHeight, ":", env.txHash)

  // encrypt notification data for this event
  let seed := getSeedFor(recipientAddr)
  let [ciphertext, tag] := chacha20poly1305_encrypt(key=seed, nonce=nonce, message=message, aad=aad)

  // concatenate ciphertext and 16 bytes of tag (note: crypto libs typically default to doing it this way in `seal`)
  let payload := concat(ciphertext, tag)

  return payload
}
```


Pseudocode for decrypting data from notifications (client):
```
fun decryptNotificationData(contractAddr, channelId, payload, env) {
  // depending on which mode the channel operates in
  if inCounterMode(channelId):
    // counter reflects the nth notification for the given recipient in the given channel
    let counter := getCounterFor(recipientAddr, channelId)

    // encode uint64 counter in BE and left-pad with 4 bytes of 0x00 to make 12 bytes
    saltBytes := concat(zeros(4), uint64BigEndian(counter))

  // otherwise, channel is in TxHash Mode
  else:
    // take first 12 bytes of tx hash (make sure to decode the hex string)
    saltBytes := slice(hexToBytes(env.txHash), 0, 12)

  // ChaCha20 expects a 96-bit (12 bytes) nonce
  // take the first 12 bytes of the channel id's sha256 hash
  let channelIdBytes := slice(sha256(utf8ToBytes(channelId)), 0, 12)

  // produce the nonce by XOR'ing the two previous 12-byte results
  let nonce := xorBytes(channelIdBytes, counterBytes)

  // construct the additional authenticated data
  let aad := concatStrings(env.blockHeight, ":", env.txHash)

  // split payload
  let ciphertext := slice(payload, 0, len(payload) - 16)
  let tag := slice(payload, len(ciphertext))

  // decrypt notification data
  let seed := getSeedFor(contractAddr)
  let message := chacha20poly1305_decrypt(key=seed, nonce=nonce, message=ciphertext, tag=tag, aad=aad)

  // do not trim trailing zeros because there is no END marker in CBOR. just decode plaintext as-is
  let plaintext := message

  return plaintext
}
```


## Dispatch Notification Algorithm

Pseudocode for dispatching a notification (contract):
```
fun dispatchNotification(recipientAddr, channelId, plaintext, env) {
  // obtain the current notification ID
  let notificationId := notificationIDFor(recipientAddr, channelId)

  // construct the notification data payload
  let payload := encryptNotificationData(recipientAddr, channelId, plaintext, env);

  // increment the counter
  incrementCounterFor(contractAddr, channelId)

  // emit the notification
  addAttributeToEventLog(notificationId, payload)
}
```



## Cryptography

A quick overview of the involved cryptographic features:


### HKDF

The contract must [derive an internal secret](#contract-internal-secret-derivation) upon initialization.

Subsequently, the contract uses its internal secret and a recipient's address to derive a unique key for that recipient without them having to execute a tx (it gets shared when they make an authenticated query for it).


### Secp256k1

Recipients can optionally establish a new shared secret with the contract to provide better security against hypothetical side-chain attacks. The contract enforces determinism and high entropy for new shared secrets by requiring users to [submit a signed document](#updateseed-method) that references the previous seed.


### HMAC-SHA256

Used to [generate Notification IDs](#notification-id-algorithm).


### ChaCha20-Poly1305

This AEAD (authenticated encryption with additional data) algorithm was selected to encrypt and authenticate notification data based on its low-cost performance profile, making very efficient use of gas, and its widespread adoption, simplifying both contract and client-side implementations.

Within the contract, implementations are advised to use RustCrypto's AEADs ([docs](https://docs.rs/chacha20poly1305/latest/chacha20poly1305/), [crate](https://crates.io/crates/chacha20poly1305), [GitHub](https://github.com/RustCrypto/AEADs)), which has been audited for usage inside SGX enclaves ([report](https://research.nccgroup.com/2020/02/26/public-report-rustcrypto-aes-gcm-and-chacha20poly1305-implementation-review/)).


## Privacy Considerations


### Padding: Constant-Length Notification Data

Contracts SHOULD pad the value of the encrypted attribute added to the event log (i.e., the [Notification Data](#notification-data)) to some constant length per channel.

For example, consider a "direct_message" channel with the following CDDL:
```cddl
direct_message = [
  id: uint,
  replying_to: uint,
  contents: text,
]
```

In CBOR, the maximum value of `uint` is 64 bits (8 bytes), and `text` (or `tstr`) does not have an inherent limit. In order to achieve constant-length notification data, we need to enforce a maximum size for the `contents` member of this tuple.

Assuming we set a maximum byte length of 128 bytes per direct message contents, we can then calculate the maximum size of a notification data's plaintext as follows:
```
  + 1 byte  ; cbor array length < 24
  + 1 byte  ; uint type
  + 8 bytes  ; id value
  + 1 bytes  ; uint type
  + 8 bytes  ; replying_to value
  + 2 bytes   ; text string length >= 24 and < 255
  + 128 bytes  ; contents value
= 149 bytes  ; plaintext size
```

Finally, in the [Notification Data Algorithms](#notification-data-algorithms) pseudocode, we would set `DATA_LEN` to `149` bytes. This ensures that the "direct_message" channel always emits a constant-length attribute value.

> NOTE: even if a channel is only using `uint`, padding still applies since CBOR will use the shortest possible encoding.


### Decoy Notifications: Constant Log Size

Contracts SHOULD emit a constant number of attributes to the event log on every execution in order to conceal which transactions emitted notification(s) versus those that didn't.

For example, if a contract employs three distinct notification channels, then every transaction should result in three key-value attributes being added to the event log (e.g., by calling `add_attribute_plaintext(...)` three times) no matter what the execution message was (and therefore no matter how many actual notifications were emitted).

When emitting decoy notifications, it is recommended to use all NULL bytes (up to `DATA_LEN`) as the notification data and the contract's own address as the recipient.

Furthermore, it is advised to emit decoy notifications as if the channel was operating in TxHash Mode, so that an attacker cannot possibly deduce that a notification is a decoy. Emitting decoy notifications in TxHash mode also has the added benefit of not needing to store/update a counter variable.


### Hypothetical Attack in Counter Mode

The following section describes a hypothetical side-chain attack for channels operating in Counter Mode in which the attacker performs a [replay attack]((https://docs.scrt.network/secret-network-documentation/overview-ecosystem-and-technology/techstack/privacy-technology/theoretical-attacks/contract-level/replay-side-chain)). However, channels operating in TxHash Mode are completely immune to this type of attack and don't require any counter-measures. Contracts should still strive to emit a consistent number of events that appear as notifications in order to mask actual events with noise.

If a contract action allows _any sender_ to trigger a notification for some recipient, then there is a risk that an attacker could perform a side-chain attack to precompute a victim's next Notification ID for a specific channel within a contract.

For example, Mallory has a balance of 0 IBC TKN on chain. She broadcasts a transaction with two messages: deposit 10 IBC TKN to the contract and then transfer 10 wrapped TKN to Alice. The execution fails since her IBC TKN balance is insufficient. Mallory then forks the chain, Cosmos Bank transfers 10 IBC TKN from another account, then replays the failed transaction on her side chain. This time, the deposit and subsequent wrapped TKN transfer succeeds since she has a sufficient balance. Mallory then records the emitted Notification ID (which belongs to Alice) and waits to observe that same Notification ID on the actual chain. At that point, Mallory would be able to deduce that _someone_ transferred _some amount_ of TKN to Alice.

Notice that the threat model looks very different if the contract only allowed friends of Alice to transfer tokens to her account (i.e., no longer _any sender_). In that case, only a friend of Alice would be able to perform the attack.

Also notice that if Alice executes the UpdateSeed method after Mallory forks the chain and before Bob transfers tokens, then Mallory's attack fails.

Again, TxHash Mode prevents this type attack, but sacrifices some of the benefits that come with predictable Notification IDs. Contract developers should consider the privacy requirements of their application when choosing which mode to use for a given channel.
