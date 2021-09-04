# SNIP-23 - Improved UX to [SNIP-20](/SNIP-20.md) Send Operations

This document describes an optional enhancement to the SNIP-20 spec. Contracts that support the existing SNIP-20 standard are still considered compliant, and clients should be written to benefit from these features when available, but provide fallback for when these features are not available.

The feature specified in this document is an improved UX for the `Send` and `SendFrom` operations, aimed at enriching SNIP20 tokens composability and usage.

- [SNIP-23 - Improved UX to SNIP-20 Send Operations](#snip-23---improved-ux-to-snip-20-send-operations)
  - [The optional `recipient_code_hash` parameter](#the-optional-recipient_code_hash-parameter)
    - [Updated `Send` interface](#updated-send-interface)
    - [Updated `SendFrom` interface](#updated-sendfrom-interface)

## The optional `recipient_code_hash` parameter

Up until now, any contract wishing to be invoked (notified) upon receiving a SNIP20 token, had to "register" its `code_hash` with that SNIP20 token. The reason is that on Secret Network due to security concerns, any caller to a contract must also know the SHA256 value of the contract's binary code. When a user sends tokens to a contract they usually don't know the contract's SHA256 value, and without it the SNIP20 contract wouldn't be able to invoke (notify) the receiving contract. Therefore, the SNIP20 spec made every contract wishing to receive a call upon receiving funds, to first register with that SNIP20 token.

In practice, getting the SHA256 value of a contract from the chain is very easy, and implementing a contract that receives SNIP20 token transfers became a bit cumbersome, because of the added boilerplate code and the need to know in advance which tokens the contract should support.

SNIP23 introduces an optional `recipient_code_hash` parameter in the `Send` and `SendFrom` operations, thus contracts are not required to "register" with the SNIP23 contract before receiving tokens.

### Updated `Send` interface

| Name                | Type             | Description                                                                                           | optional |
| ------------------- | ---------------- | ----------------------------------------------------------------------------------------------------- | -------- |
| recipient           | string           | Accounts SHOULD be a valid bech32 address, but contracts may use a different naming scheme as well    | no       |
| recipient_code_hash | string           | A lower-case SHA256 hash of the recipient contract's code. Must have 64 characters (No leading "0x"). | yes      |
| amount              | string (Uint128) | The amount of tokens to send                                                                          | no       |
| msg                 | string (base64)  | Base64 encoded message, which the recipient will receive                                              | yes      |
| memo                | string           | A private message only the sender and recipient can see                                               | yes      |
| padding             | string           | Ignored string used to maintain constant-length messages                                              | yes      |

### Updated `SendFrom` interface

| Name                | Type             | Description                                                                                           | optional |
| ------------------- | ---------------- | ----------------------------------------------------------------------------------------------------- | -------- |
| owner               | string           | Account to take tokens **from**                                                                       | no       |
| recipient           | string           | Account to send tokens **to**                                                                         | no       |
| recipient_code_hash | string           | A lower-case SHA256 hash of the recipient contract's code. Must have 64 characters (No leading "0x"). | yes      |
| amount              | string (Uint128) | Amount of tokens to transfer                                                                          | no       |
| msg                 | string (base64)  | Base64 encoded message, which the recipient will receive                                              | yes      |
| memo                | string           | A private message only the sender and recipient can see                                               | yes      |
| padding             | string           | Ignored string used to maintain constant-length messages                                              | yes      |
