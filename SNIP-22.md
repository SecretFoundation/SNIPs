# SNIP-22 - Batch Operations for [SNIP-20](/SNIP-20.md) Contracts

This document describes several optional enhancements to the SNIP-20 spec.
Contracts that support the existing SNIP-20 standard are still considered
compliant, and clients should be written so as to benefit from these features
when available, but provide fallbacks for when these features are not available.

The features specified in this document are batched versions of the existing
`Transfer`, `Send`, `Mint`, `TransferFrom`, and `SendFrom` messages.

* [BatchTransfer](#BatchTransfer)
* [BatchSend](#BatchSend)
* [BatchTransferFrom](#BatchTransferFrom)
* [BatchSendFrom](#BatchSendFrom)
* [BatchMint](#BatchMint)

## BatchTransfer
Moves tokens from the account that appears in the Cosmos message sender field
to the accounts in the recipient field.

* Variation from CW-20: It is NOT required to validate that the recipient is an
  address and not a contract. This command will work when trying to send funds
  to contract accounts as well.

### Request
Accounts SHOULD be a valid bech32 address, but
contracts may use a different naming scheme as well.

* `.actions[].recipient` - Account to take tokens __to__
* `.actions[].amount` - The amount of tokens to transfer as a Uint128
  (integer string)
* `.padding` - Optional ignored string used to maintain constant-length messages

Example:
```json
{
  "batch_transfer": {
    "actions": [
      { "recipient": "secret1", "amount": "123" },
      { "recipient": "secret1", "amount": "123" },
      { "recipient": "secret1", "amount": "123" }
    ],
    "padding": "this string is ignored"
  }
}
```

### Response
```json
{
  "batch_transfer": {
    "status": "success"
  }
}
```

## BatchSend
Moves amount from the Cosmos message sender account to the recipient accounts.
The receiver accounts MAY be contracts that have registered themselves using a
[RegisterReceive](./SNIP-20.md#RegisterReceive) message. If such a registration has been
performed, a message MUST be sent to the contract's address as a callback,
after completing the transfer. The format of this message is described under
[Receiver interface](./SNIP-20.md#receiver-interface).

If the callback fails due to an error in the Receiver contracts, the entire
transaction will be reverted.

### Request
Accounts SHOULD be a valid bech32 address, but
contracts may use a different naming scheme as well.

* `.actions[].recipient` - Account to take tokens __to__
* `.actions[].amount` - The amount of tokens to transfer as a Uint128
  (integer string)
* `.actions[].msg` - Base64 encoded message, which the recipient will receive
* `.padding` - Optional ignored string used to maintain constant-length messages

Example:
```json
{
  "batch_send": {
    "actions": [
      { "recipient": "secret1", "amount": "123", "msg": "Zm9vYmFyYmF6Cg==" },
      { "recipient": "secret1", "amount": "123", "msg": "Zm9vYmFyYmF6Cg==" },
      { "recipient": "secret1", "amount": "123", "msg": "Zm9vYmFyYmF6Cg==" }
    ],
    "padding": "this string is ignored"
  }
}
```

### Response
```json
{
  "batch_send": {
    "status": "success"
  }
}
```

## BatchTransferFrom
Transfer an amount of tokens from a specified accounts, to other specified
accounts. This action MUST fail if the Cosmos message sender does not have an
_allowance_ limit that is equal or greater than the amount of tokens sent for
the  _owner_ accounts

### Request
Accounts SHOULD be a valid bech32 address, but
contracts may use a different naming scheme as well.

* `.actions[].owner` - Account to take tokens __from__
* `.actions[].recipient` - Account to take tokens __to__
* `.actions[].amount` - The amount of tokens to transfer as a Uint128
  (integer string)
* `.padding` - Optional ignored string used to maintain constant-length messages

Example:
```json
{
  "batch_transfer_from": {
    "actions": [
      { "owner": "secret1", "recipient": "secret1", "amount": "123" },
      { "owner": "secret1", "recipient": "secret1", "amount": "123" },
      { "owner": "secret1", "recipient": "secret1", "amount": "123" }
    ],
    "padding": "this string is ignored"
  }
}
```

### Response
```json
{
  "batch_transfer_from": {
    "status": "success"
  }
}
```

## BatchSendFrom
BatchSendFrom is to BatchSend, what TransferFrom is to Transfer. This allows a
pre-approved account to not just transfer the tokens,
but to send them to other addresses to trigger a given action.
Note BatchSendFrom will set the `Receive::{sender}` to be the
`env.message.sender` (the account that triggered the transfer) rather than
the owner account (the account the money is coming from).

### Request
Accounts SHOULD be a valid bech32 address, but
contracts may use a different naming scheme as well.

* `.actions[].owner` - Account to take tokens __from__
* `.actions[].recipient` - Account to take tokens __to__
* `.actions[].amount` - The amount of tokens to transfer as a Uint128
  (integer string)
* `.actions[].msg` - Base64 encoded message, which the recipient will receive
* `.padding` - Optional ignored string used to maintain constant-length messages

Example:
```json
{
  "batch_transfer_from": {
    "actions": [
      { "owner": "secret1", "recipient": "secret1", "amount": "123", "msg": "Zm9vYmFyYmF6Cg==" },
      { "owner": "secret1", "recipient": "secret1", "amount": "123", "msg": "Zm9vYmFyYmF6Cg==" },
      { "owner": "secret1", "recipient": "secret1", "amount": "123", "msg": "Zm9vYmFyYmF6Cg==" }
    ],
    "padding": "this string is ignored"
  }
}
```

### Response
```json
{
  "batch_send_from": {
    "status": "success"
  }
}
```

## BatchMint
This function MUST be allowed only for accounts on the minters list.

If the Cosmos message sender is an allowed minter, this will create new tokens
in the balances of the listed accounts, matching the amount listed in each
action.

### Request
Accounts SHOULD be a valid bech32 address, but
contracts may use a different naming scheme as well.

* `.actions[].recipient` - Account to take tokens __to__
* `.actions[].amount` - The amount of tokens to transfer as a Uint128
  (integer string)
* `.padding` - Optional ignored string used to maintain constant-length messages

Example:
```json
{
  "batch_mint": {
    "actions": [
      { "recipient": "secret1", "amount": "123" },
      { "recipient": "secret1", "amount": "123" },
      { "recipient": "secret1", "amount": "123" }
    ],
    "padding": "this string is ignored"
  }
}
```

### Response
```json
{
  "batch_mint": {
    "status": "success"
  }
}
```

