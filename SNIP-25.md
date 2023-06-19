# SNIP-25 - Improved privacy for [SNIP-20](/SNIP-20.md) tokens

This document describes enhancements to the SNIP-20 spec, namely, the addition of `decoys` and `entropy` in execution messages in order to mitigate storage access pattern attacks. Decoys are an optional reference to a vector of Addr which, when executed with sufficient entropy, add an additional layer of privacy by obfuscating the real accounts that are being updated.

Contracts that support the existing SNIP-20 standard are still considered
compliant, and clients SHOULD be written so as to benefit from this feature
when available.

- [Rationale](#rationale)
- [Messages](#messages)
- [Batch Operations](#batch-operations)
- [Randomness Appendix](#randomness-appendix)

## Rationale

Secret Network smart contracts provide computational privacy through the use of Trusted Execution Environments, which process encrypted data securely without revealing it publicly on chain. However, the patterns at which data is accessed when processing confidential transactions can leak crucial information. In the example of a SNIP-20, each account balance may be stored privately, but the access pattern is public, so an untrusted host OS can potentially learn private information such as recipient addresses by monitoring accesses to the key-value storage.

Because the same key is used each time a user's balance is accessed, it's possible to link two transactions together. Let's say someone sends tokens from address A to address B, then later someone sends tokens from address B to address C. By knowing the keys involved, one can work out that the same address was involved in both transactions - effectively 'unmasking' the private recipient.

Decoy addresses with sufficient randomness can be used to mitigate storage access pattern leakage by obfuscating the actual address that is the recipient of funds in the contract's storage. Decoys SHOULD be used for every message exposed to storage access pattern attacks, and for every transaction that does it MUST include the `decoys` parameter. Execution messages which accept the `decoys` field MAY also accept the `entropy` field, which is an optional Binary used to introduce randomness by providing additional read and writes to the decoy addresses that are included in the message. How `entropy` is implemented is left to the developer's discretion. For example, one MAY include the `decoys` field and not the `entropy` field and use Secret VRF for randomness.

## Messages

The existing `Redeem`, `Deposit`, `Transfer`, `Send`, `Burn`, `TransferFrom`, `SendFrom`, `BurnFrom`, and `Mint` messages now may also accept `decoy` and `entropy` fields.

Example `Transfer` message, with optional reference to `decoys` included.

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

Batch operations are used for handling multiple token transfers or operations in a single transaction. In the case of batch messages, the `decoys` parameter SHOULD be included with the [actions](https://github.com/scrtlabs/snip20-reference-impl/blob/ea9fb0cd76f3e0d48e86b4d02a3990f2f4a84d00/src/batch.rs#LL34C1-L40C2), rather than the messages themselves:

Batch Transfer Message:

```rust
BatchTransfer {
        actions: Vec<batch::TransferAction>,
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

## Randomness Appendix

How `entropy` is implemented is left to the developer's discretion.

- [See here](https://github.com/scrtlabs/snip20-reference-impl/blob/ea9fb0cd76f3e0d48e86b4d02a3990f2f4a84d00/src/state.rs#L136) for an example entropy implementation in a snip-20 compliant contract.
- [See here](https://docs.scrt.network/secret-network-documentation/development/development-concepts/randomness-api/native-on-chain-randomness) to learn how to use Secret VRF for randomness.
