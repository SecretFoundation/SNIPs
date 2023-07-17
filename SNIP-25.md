# SNIP-25 - Improved privacy for [SNIP-20](/SNIP-20.md) tokens

This document outlines updates to the SNIP-20 specification with the introduction of `decoys`. A decoy, in this context, is a false or 'dummy' account that is intermingled with real accounts during transactions. The main function of these decoys is to act as distractions or misdirections to potential attackers, thereby making it more difficult to pinpoint the actual accounts involved in a transaction.

The concept of decoys is designed to strengthen privacy measures by ensuring that account information remains concealed during transaction processes. By intermixing real accounts with these 'decoy' accounts, a level of uncertainty and complexity is created, which can deter potential attackers trying to discern the actual accounts being updated. Additionally, randomness is integrated into the execution messages, providing an extra layer of protection by creating uncertainty in storage access patterns.

## Notation

"Should", "May", and "Must", notation in this document conforms to the standards set by
[RFC-2119](http://microformats.org/wiki/rfc-2119)

Contracts that support the existing SNIP-20 standard are still considered
compliant, and clients SHOULD be written so as to benefit from this feature
when available.

- [Rationale](#rationale)
- [Messages](#messages)
- [Batch Operations](#batch-operations)
- [Randomness Appendix](#randomness-appendix)

## Rationale

Secret Network smart contracts provide computational privacy through the use of Trusted Execution Environments, which process encrypted data securely without revealing it publicly on chain. However, the patterns at which data is accessed when processing confidential transactions can leak crucial information. In the example of a SNIP-20, each account balance may be stored privately, but the access pattern is public, so an untrusted host OS can potentially learn private information by monitoring accesses to the key-value storage. For example, each user's balance is kept in a key-value store, and each account is associated with a consistent, encrypted key. By keeping track of the keys fetched from the storage, an outside observer can decipher the receiver of a SNIP-20 transfer.

![Storage Access Pattern](https://i.ibb.co/N1JsQrC/storage-access-pattern.png)

In order to effectively incorporate decoys, it is necessary to randomly interweave decoy addresses with the real addresses, ensuring that the real addresses are included within the sequence. Decoys SHOULD be used for every message exposed to storage access pattern attacks, and for every transaction that does, it MUST include the `decoys` parameter. Provided decoy addresses MUST be treated and utilized as if they were genuine addresses and MUST be used in a way that is indistinguishable from real addresses. Execution messages which accept the `decoys` field MAY also accept the `entropy` field, which is an optional Binary used to introduce randomness by providing additional read and writes to the decoy addresses that are included in the message. How `entropy` is implemented is left to the developer's discretion. For example, one MAY include the `decoys` field and not the `entropy` field and use Secret VRF for randomness.

## Messages

The existing `Redeem`, `Deposit`, `Transfer`, `Send`, `Burn`, `TransferFrom`, `SendFrom`, `BurnFrom`, and `Mint` messages now SHOULD accept the `decoys` field.

Example `Transfer` message with `decoys` field included, and optional `entropy` field omitted (How randomness is implemented is left to the developer's discretion).

##### Request

```json
{
  "transfer": {
    "recipient": "secret1jxlkaddg4nvsvnjylx94xnwrjprqvvp20r4hxd",
    "amount": "1000000000000000000",
    "memo": "Transfer for services rendered",
    "decoys": [
      "secret1zpaxzv6yggvmj4jj7md94rjxvnpa6q07asxyyh",
      "secret12g3ujhpkunstpdg78f8fny0r59zg6e9kghgpxy"
    ]
  }
}
```

##### Response

```json
{
  "transfer": {
    "status": "success"
  }
}
```

## Batch Operations

[Batch operations](./SNIP-22.md) are used for handling multiple token transfers or operations in a single transaction. In the case of batch messages, the `decoys` parameter SHOULD be defined in the [actions](https://github.com/scrtlabs/snip20-reference-impl/blob/ea9fb0cd76f3e0d48e86b4d02a3990f2f4a84d00/src/batch.rs#LL34C1-L40C2) module and imported into the [messages](https://github.com/scrtlabs/snip20-reference-impl/blob/ea9fb0cd76f3e0d48e86b4d02a3990f2f4a84d00/src/msg.rs#L188) module:

Batch Transfer Message:

```rust
BatchTransferFrom {
        actions: Vec<batch::TransferFromAction>,
        entropy: Option<Binary>,
        padding: Option<String>,
    }
```

Batch Transfer Action:

```rust
pub struct TransferFromAction {
    pub owner: String,
    pub recipient: String,
    pub amount: Uint128,
    pub memo: Option<String>,
    pub decoys: Option<Vec<Addr>>,
}
```

Refer to [SNIP-22](./SNIP-22.md) for further reading on batch operations.

## Randomness Appendix

How randomness is implemented is left to the developer's discretion.

- [See here](https://github.com/scrtlabs/snip20-reference-impl/blob/ea9fb0cd76f3e0d48e86b4d02a3990f2f4a84d00/src/state.rs#L136) for an example entropy implementation in a snip-20 compliant contract.
- [See here](https://docs.scrt.network/secret-network-documentation/development/development-concepts/randomness-api/native-on-chain-randomness) to learn how to use Secret VRF for randomness.
