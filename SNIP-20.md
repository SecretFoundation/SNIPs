# SNIP-20 Spec: Private, Fungible Tokens
SNIP-20, is a specification for private fungible tokens based on
CosmWasm on the Secret Network. The name and design is loosely based on
Ethereum's [ERC-20](https://eips.ethereum.org/EIPS/eip-20) &
[ERC-777](https://eips.ethereum.org/EIPS/eip-777) standards, and a superset of
CosmWasm's [CW-20](https://github.com/CosmWasm/cosmwasm-plus/blob/master/packages/cw20/README.md).
The additions to this spec over the CW20 specification are mostly for privacy
focused features, and as such will strive to maintain compatibility.

The specification is split into multiple sections, a contract may only implement
some of this functionality, but must implement the base functionality.

Contracts that implement this interface SHOULD support only a single
token/symbol, and MUST conform to the standards set in this memo.

* [Introduction](#Introduction)
* [Sections](#Sections)
    * [Base](#Base)
    
        Messages:
        * [Transfer](#Transfer)
        * [Send](#Send)
        * [RegisterReceive](#RegisterReceive)
        * [CreateViewingKey](#CreateViewingKey)
        * [SetViewingKey](#SetViewingKey)
  
        Queries:
        * [Balance](#Balance)
        * [TokenInfo](#TokenInfo)
        * [TransferHistory](#TransferHistory)
    
    * [Allowances](#Allowances)
        
        Messages:
        * [IncreaseAllowance](#IncreaseAllowance)
        * [DecreaseAllowance](#DecreaseAllowance)
        * [TransferFrom](#TransferFrom)
        * [SendFrom](#SendFrom)
        
        Queries:
        * [Allowance](#Allowance)
    
    * [Mintable](#Mintable)
        
        Messages:
        * [Mint](#Mint)
        * [SetMinters](#SetMinters)
        * [Burn](#Burn)
        * [BurnFrom](#BurnFrom)
        
        Queries:
        * [Minters](#Minters)
    
    * [Native](#Native)
    
        Messages:
        * [Deposit](#Deposit)
        * [Redeem](#Redeem)
        
        Queries:
        * [ExchangeRate](#ExchangeRate)

* [Receiver interface](#Receiver-interface)

# Introduction

## Scope
This document aims to set standard interfaces that SNIP-20 contract
implementors will create, and that both wallet implementors & dependant contract
creators will consume.
For this reason, the focus of this document is to merely give SNIP-20
contract implementors the tools needed to create contracts that fully maintain
privacy but not to specify implementation details.

## Notation
Notation in this document conforms to the standards set by
[RFC-2119](http://microformats.org/wiki/rfc-2119)

## Terms
- __Message__ - This is an on-chain interface. It is triggered by sending a
  transaction, and receiving an on-chain response which is read by the client.
  Messages are authenticated both by the blockchain, and by the secret enclave.
- __Query__ - This is an off-chain interface. Queries are done by returning data
  that a node has locally, and are not public. Query responses are returned
  immediately, and do not have to wait for blocks. In addition, queries cannot
  be authenticated using the standard interfaces. Any contract that wishes to
  strongly enforce query permissions must implement it themselves. (TBD -
  should this be standardized?)
- __Cosmos Message Sender__ - The account that is found under the `sender`
  field in a standard Cosmos SDK message. This is also the signer of the message.
- __Native Asset__ - A coin which is defined and managed by the blockchain
  infrastructure, not a secret contract.

## Padding
Users may want to enforce constant length messages to avoid leaking
data. To support this functionality, SNIP-20 tokens MUST support the option to
include a _padding_ field in every message. This optional _padding_ field may be
sent with ANY of the messages in this spec. Contracts and Clients MUST ignore
this field if sent, either in the request or response fields

## Requests
Requests SHOULD be sent as base64 encoded JSON. Future versions of Secret
Network may add support for other formats as well, but at this time we recommend
usage of JSON only. For this reason the parameter descriptions specify the JSON
type which must be used. In addition, request parameters will include in
parentheses a CosmWasm (or other) underlying type that this value must conform
to. E.g. a recipient address is sent as a string, but must also be parsed to a
bech32 address.

## Queries
Queries are off-chain requests, that are not cryptographically validated. This
means that contracts that wish to validate the caller of a query MUST implement
some sort of authentication. SNIP-20 uses an "API key" scheme, which validates
a `(viewing key, account)` pair.

Authentication MUST happen on each query that reveals private account-specific
information.
Authentication MUST be a resource intensive operation, that takes a significant
amount of time to compute. This is because such queries are open to offline
brute-force attacks, which can be parallelized to scale linearly with the
resources of a motivated attacker.
Authentication MUST perform the same computation even if the user does not have
a viewing key set.
Authentication response MUST be indistinguishable for both the case of a wrong
viewing key and the case of a non-existent viewing key.

## Responses
Unless specified otherwise, all message & query responses will be JSON encoded
in the `data` field of the Cosmos response, rather than in the `logs`.
This is meant to reduce the potential for data-leakage through side-channel
attacks. In addition, since all keys will be encrypted, it is not possible to
use the `log` events for event triggering.

### Success status
Some of the messages detailed in this document contain a `"status"` field.
This field MUST hold one of two values: `"success"` or `"failure"`.

While errors during execution of contract functions should usually result in a
proper and detailed error response, The `"failure"` status is reserved for cases
where a contract might choose to obfuscate the exact cause of failure, or
otherwise indicate that while nothing failed to happen, the operation itself
could not be completed for some valid reason.

## Balances and amounts
Note that all amounts are represented as numerical strings (The Uint128 type).
Handling decimals is left to the UI.

# Sections

## Base

### Messages

#### Transfer
Moves tokens from the account that appears in the Cosmos message sender field
to the account in the recipient field.

* Variation from CW-20: It is NOT required to validate that the recipient is an
address and not a contract. This command will work when trying to send funds to
contract accounts as well.

##### Request
|Name      |Type             |Description                                                                                                 | optional |
|----------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|recipient | string          |  Accounts SHOULD be a valid bech32 address, but contracts may use a different naming scheme as well        |  no      |
|amount    | string (Uint128)|  The amount of tokens to transfer                                                                          |  no      |
|padding   | string          |  Ignored string used to maintain constant-length messages                                                  |  yes     |

##### Response
```json
{
  "transfer": {
    "status": "success"
  }
}
```

#### Send
Moves amount from the Cosmos message sender account to the recipient account.
The receiver account MAY be a contract that has registered itself using a
[RegisterReceive](#RegisterReceive) message. If such a registration has been
performed, a message MUST be sent to the contract's address as a callback,
after completing the transfer. The format of this message is described under
[Receiver interface](#receiver-interface).

If the callback fails due to an error in the Receiver contract, the entire
transaction will be reverted.

###### Request
|Name      |Type             |Description                                                                                                 | optional |
|----------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|recipient | string          |  Accounts SHOULD be a valid bech32 address, but contracts may use a different naming scheme as well        | no       |
|amount    | string (Uint128)|  The amount of tokens to send                                                                              | no       |
|msg       | string (base64) |  Base64 encoded message, which the recipient will receive                                                  | yes      |
|padding   | string          |  Ignored string used to maintain constant-length messages                                                  | yes      |

###### Response
```json
{
  "send": {
    "status": "success"
  }
}
```

#### RegisterReceive
This message is used to tell the SNIP-20 contract to call the `Receive` function
of the Cosmos message sender after a successful `Send`.

In Secret Network this is used to pair a code hash with the contract address
that must be called. This means that the SNIP-20 MUST store the sent code_hash
and use it when calling the `Receive` function.

##### Request
|Name         |Type             |Description                                                                                                 | optional |
|-------------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|code_hash    | string          |  A 32-byte hex encoded string, with the code hash of the receiver contract                                 | no       |
|padding      | string          |  Ignored string used to maintain constant-length messages                                                  | yes      |

##### Response
```json
{
  "register_receive": {
    "status": "success"
  }
}
```

#### CreateViewingKey
This function generates a new viewing key for the Cosmos message sender, which
is used in ALL account specific queries. This key is used to validate the
identity of the caller, since in queries in Cosmos there is no way to
cryptographically authenticate the querier's identity.

The Viewing Key MUST be stored in such a way that lookup takes a significant
amount to time to perform, in order to be resistant to brute-force attacks.
The viewing key MUST NOT control any functions which actively affect the balance
of the user.

The `entropy` field of the request should be a client supplied string used for
entropy for generation of the viewing key. Secure implementation is left to the
client, but it is recommended to use base-64 encoded random bytes and not
predictable inputs.

##### Request
|Name         |Type             |Description                                                  | optional |
|-------------|-----------------|-------------------------------------------------------------|----------|
|entropy      | string          |  A source of random information                             | no       |
|padding      | string          |  Ignored string used to maintain constant-length messages   | yes      |

##### Response
```json
{
  "create_viewing_key": {
    "key": "<string>"
  }
}
```

#### SetViewingKey
Set a viewing key with a predefined value for Cosmos message sender, _without_
creating it. This is useful to manage multiple SNIP-20 tokens using the same
viewing key.

If a viewing key is already set, the contract MUST replace the current key.
If a viewing key is not set, the contract MUST set the provided key as the
viewing key.

It is NOT RECOMMENDED to use this function to create easy to remember passwords
for users, but this is left up to implementors to enforce.

##### Request
|Name         |Type             |Description                                                                                                 | optional |
|-------------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
| key         | string          |  A user supplied string that will be used to authenticate the sender                                       | no       |
| padding     | string          |  Ignored string used to maintain constant-length messages                                                  | yes      |

##### Response
```json
{
  "set_viewing_key": {
    "status": "success"
  }
}
```

### Queries

#### Balance
This query MUST be authenticated.

Returns the balance of the given address. Returns "0" if the address is unknown
to the contract.

##### Request
|Name         |Type             |Description                                                                                                 | optional |
|-------------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
| key         | string          |  The viewing key                                                                                           | no       |
| address     | string          |  Addresses SHOULD be a valid bech32 address, but contracts may use a different naming scheme as well       | no       |

##### Response
```json
{
  "balance": {
    "amount": "123"
  }
}
```

#### TokenInfo
This query need not be authenticated.

Returns the token info of the contract. The response MUST contain: token name,
token symbol, and the number of decimals the token uses. The response MAY
additionally contain the total-supply of tokens. This is to enable Layer-2
tokens which want to hide the amounts converted as well.

##### Request
No parameters required.

##### Response
```json
{
  "name": "coin name",
  "symbol": "coin symbol",
  "decimals": 6,
  "total_supply": "Uint128"
}
```

#### TransferHistory
This query MUST be authenticated.

This query SHOULD return a list of json objects describing the transactions
made by the querying address, in newest-first order. The user may optionally
specify a limit on the amount of information returned by paging the available
items.

The response should look as described below. The `id` field is optional, and may
be used by contracts that choose to populate it. The `from` and `sender` fields
may be different if the TX was generated by a [TransferFrom](#TransferFrom)
or [SendFrom](#SendFrom) operation.

##### Request
|Name         |Type             |Description                                                                                                  | optional |
|-------------|-----------------|-------------------------------------------------------------------------------------------------------------|----------|
| key         | string          | The viewing key                                                                                             | no       |
| address     | string          | Addresses SHOULD be a valid bech32 address, but contracts may use a different naming scheme as well         | no       |
| page_size   | number          | Number of transactions to return, starting from the latest. i.e. n=1 will return only the latest transaction| no       |
| page        | number          | This defaults to 0. Specifying a positive number will skip `page * page_size` txs from the start.           | yes      |

##### Response
```json
{
  "transfer_history": {
    "txs": [
      {
        "id": "optional ID",
        "from": "address of the owner of the funds",
        "sender": "address of the sender",
        "receiver": "address of the receiver",
        "coins": {
          "denom": "coin denomination/name",
          "amount": "Uint128"
        }
      },
      { "...": "..." }
    ]
  }
}
```

## Allowances
A SNIP-20 contract may allow accounts to delegate some of their balance to
other accounts. This is very similar to the allowance feature of ERC-20
contracts. It can be used by accounts to allow other contracts to manage a
portion of their balance.

This is also useful for maintaining a higher degree of transactional privacy.
A user may choose to operate two accounts: One would deposit/withdraw funds
to the SNIP-20 contract, and then give allowance to a second account. That
second account will seem unrelated to the depositing account to outside
observers, and will be able to make transactions in its name. The recipients
of the transfer will still be able to see the connection, but the sender can
change the account that gets allowance as often as they need to. This will
allow you to minimize the interaction of the depositing account with the
contract, and dissociate the (public) calls to the contract from the funds they
operate on.

There was an issue with race conditions in the original ERC20 approval spec. If
you had an approval of 50 and I then want to reduce it to 20, I submit a Tx to
set the allowance to 20. If you see that and immediately submit a tx using the
entire 50, you then get access to the other 20. Not only did you quickly spend
the 50 before I could reduce it, you get another 20 for free.

The solution discussed in the Ethereum community was an IncreaseAllowance and
DecreaseAllowance operator (instead of Approve). To originally set an approval,
use IncreaseAllowance, which works fine with no previous allowance.
DecreaseAllowance is meant to be robust, that is if you decrease by more than
the current allowance (eg. the user spent some in the middle), it will just
round down to 0 and not make any underflow error.

### Messages

#### IncreaseAllowance
Set or increase the allowance such that spender may access up to 
`current_allowance + amount` tokens from the Cosmos message sender account.
This may optionally come with an expiration time, which if set limits when the
approval can be used (by time).

##### Request
|Name         |Type             |Description                                                                                                   | optional |
|-------------|-----------------|--------------------------------------------------------------------------------------------------------------|----------|
|spender      | string          |  The address of the account getting access to the funds                                                      | no       |
|amount       | string(Uint128) |  The number of tokens to increase allowance by                                                               | no       |
|expiration   | number          |  Time at which the allowance expires.<br>Counts the number of seconds from epoch, 1.1.1970 encoded as uint64 | yes      |
|padding      | string          |  Ignored string used to maintain constant-length messages                                                    | yes      |

##### Response
```json
{
  "increase_allowance": {
    "spender": "<address>",
    "owner": "<address>",
    "allowance": "<current allowance>"
  }
}
```

#### DecreaseAllowance
Decrease or clear the allowance by a sent amount. This may optionally come with
an expiration time, which if set limits when the approval can be used. If amount
is equal or greater than the current allowance, this action MUST set the
allowance to zero, and return a `"success"` response.

##### Request
|Name         |Type             |Description                                                                                                   | optional |
|-------------|-----------------|--------------------------------------------------------------------------------------------------------------|----------|
|spender      | string          |  The address of the account getting access to the funds                                                      | no       |
|amount       | string(Uint128) |  The number of tokens to decrease allowance by                                                               | no       |
|expiration   | number          |  Time at which the allowance expires.<br>Counts the number of seconds from epoch, 1.1.1970 encoded as uint64 | yes      |
|padding      | string          |  Ignored string used to maintain constant-length messages                                                    | yes      |

##### Response
```json
{
  "decrease_allowance": {
    "spender": "<address>",
    "owner": "<address>",
    "allowance": "<current allowance>"
  }
}
```

#### TransferFrom
Transfer an amount of tokens from a specified account, to another specified
account. This action MUST fail if the Cosmos message sender does not have an
_allowance_ limit that is equal or greater than the amount of tokens sent for
the  _owner_ account

##### Request
|Name         |Type             |Description                                                  | optional |
|-------------|-----------------|-------------------------------------------------------------|----------|
|owner        | string          |  Account to take tokens __from__                            | no       |
|recipient    | string          |  Account to send tokens __to__                              | no       |
|amount       | string(Uint128) |  Amount of tokens to transfer                               | no       |
|padding      | string          |  Ignored string used to maintain constant-length messages   | yes      |

##### Response
```json
{
  "transfer_from": {
    "status": "success"
  }
}
```

#### SendFrom
SendFrom is to Send, what TransferFrom is to Transfer. This allows a
pre-approved account to not just transfer the tokens,
but to send them to another address to trigger a given action. Note SendFrom
will set the `Receive{sender}` to be the `env.message.sender`
(the account that triggered the transfer) rather than the owner account (the
account the money is coming from).

##### Request
|Name         |Type             |Description                                               | optional |
|-------------|-----------------|----------------------------------------------------------|----------|
|owner        | string          | Account to take tokens __from__                          | no       |
|recipient    | string          | Account to send tokens __to__                            | no       |
|amount       | string(Uint128) | Amount of tokens to transfer                             | no       |
|msg          | string(base64)  | Base64 encoded message, which the recipient will receive | yes      |
|padding      | string          | Ignored string used to maintain constant-length messages | yes      |

##### Response
```json
{
  "send_from": {
    "status": "success"
  }
}
```

### Queries

#### Allowance
This query MUST be authenticated.

This returns the available allowance that spender can access from the owner's
account, along with the expiration info.

Every account's viewing key MUST be given permissions to query the allowance
of any pair of owner and spender, as long as that account is either the owner
or the spender in the query. In other words, every account's viewing key can
be used to find out how much allowance the account has given other accounts,
and how much it has been given by other accounts.

The `expiration` field of the response may be either `null` or unset if no
expiration has been set.

##### Request
|Name         |Type             |Description                                                         | optional |
|-------------|-----------------|--------------------------------------------------------------------|----------|
|owner        | string          | Account from which tokens are allowed to be taken                  | no       |
|spender      | string          | Account which is allowed to spend tokens on behalf of the _owner_  | no       |
|key          | string          | The viewing key                                                    | no       |

##### Response
```json
{
  "allowance": {
    "spender": "<address>",
    "owner": "<address>",
    "allowance": "<current allowance>",
    "expiration": 1234
  }
}
```

## Mintable
This allows another contract to mint new tokens, possibly with a cap.
There is only one minter specified here, if you want more complex access management.
please use a multisig or other contract as the minter address and handle updating the ACL there.

The functions in this section allow managing a list of addresses that have
permissions to mint new coins and increase the total supply. 
You may choose to only give this permission to only one address, or
a multisig address, but we leave the option open for some contracts to define
more than one address, as either a backup, or because they choose to trust
more than one party.

Access to these functions should normally be restricted to a very small set of
known addresses (e.g. an predetermined admin address, or members of
the minters list).

### Messages

#### Mint
This function MUST be allowed only for accounts on the minters list.

If the Cosmos message sender is an allowed minter, this will create `amount` new
tokens and add them to the balance of `recipient`.

##### Request
|Name         |Type             |Description                                                 | optional |
|-------------|-----------------|------------------------------------------------------------|----------|
|recipient    | string          |  Account to mint tokens __to__                             | no       |
|amount       | string(Uint128) |  Amount of tokens to mint                                  | no       |
|padding      | string          |  Ignored string used to maintain constant-length messages  | yes      |

##### Response
```json
{
  "mint": {
    "status": "success"
  }
}
```

#### SetMinters
This function MUST only be allowed for authorized accounts.

The list of addresses in the message will be set to the list of minters
in the contract. This completely overrides the previously saved list.

##### Request
|Name         |Type             |Description                                                  | optional |
|-------------|-----------------|-------------------------------------------------------------|----------|
|minters      | list of strings |  List of addresses to set to the list of minters            | no       |
|padding      | string          |  Ignored string used to maintain constant-length messages   | yes      |

##### Response
```json
{
  "set_minters": {
    "status": "success"
  }
}
```

#### Burn
MUST remove `amount` tokens from the balance of the Cosmos message sender and
MUST reduce the total supply by the same amount.
MUST NOT transfer the funds to another account.

##### Request
|Name      |Type             |Description                                                 | optional |
|----------|-----------------|------------------------------------------------------------|----------|
|amount    | string (Uint128)|  The amount of tokens to burn                              | no       |
|padding   | string          |  Ignored string used to maintain constant-length messages  | yes      |

##### Response
```json
{
  "burn": {
    "status": "success"
  }
}
```

#### BurnFrom
This works like TransferFrom, but burns the tokens instead of transferring them.
This will reduce the owner's balance, total_supply and the caller's allowance.

This function should be available when a contract supports both the
[Mintable](#Mintable) and [Allowances](#Allowances) interfaces.

##### Request
|Name         |Type             |Description                                                | optional |
|-------------|-----------------|-----------------------------------------------------------|----------|
|owner        | string          |  Account to take tokens __from__                          | no       |
|amount       | string(Uint128) |  Amount of tokens to burn                                 | no       |
|padding      | string          |  Ignored string used to maintain constant-length messages | yes      |

##### Response
```json
{
  "burn_from": {
    "status": "success"
  }
}
```

### Queries

#### Minters
Returns the list of minters that have been configured in the contract.

##### Request
None

##### Response
```json
{
  "minters": {
    "minters": ["<address>", "<address>", "<address>"]
  }
}
```

## Native
This section describes functions that can be useful for coins that are backed
by some other _native asset_. Using these, you can create privacy tokens which
wrap assets, such as SCRT, ATOM, etc.

### Messages

#### Deposit
Deposits a native coin into the contract, which will mint an equivalent amount
of tokens to be created. The amount MUST be sent in the `sent_funds` field of
the transaction itself, as coins must really be sent to the contract's native
address.
The minted amounts MUST match the exchange rate specified by the
[ExchangeRate](#ExchangeRate) query. 
The deposit MUST return an error if any coins that do not match expected
denominations are sent.

##### Request
|Name        |Type             |Description                                                | optional |
|------------|-----------------|-----------------------------------------------------------|----------|
|padding     | string          |  Ignored string used to maintain constant-length messages | yes      |

##### Response
```json
{
  "deposit": {
    "status": "success"
  }
}
```

#### Redeem
Redeems tokens in exchange for native coins. The redeemed tokens SHOULD be
burned, and taken out of the pool.

##### Request
|Name         |Type             |Description                                                                     | optional |
|-------------|-----------------|--------------------------------------------------------------------------------|----------|
|amount       | string(Uint128) |  The amount of tokens to redeem __to__                                         | no       |
|denom        | string          |  Denom of tokens to mint. Only used if the contract supports multiple denoms   | yes      |
|padding      | string          |  Ignored string used to maintain constant-length messages                      | yes      |

##### Response
```json
{
  "redeem": {
    "status": "success"
  }
}
```

### Queries

#### ExchangeRate
This query need not be authenticated.

Gets information about the token exchange rate functionality that the contract
provides.
This query MUST return.
* exchange rate, as an integer string. The amount of native coins that equal one
  token.
* Denomination of native tokens which are acceptable, as a string OR a comma
  separated value.

##### Request
None

##### Response
```json
{
  "exchange_rate": {
    "rate": "<U128>",
    "denom": "<denom>"
  }
}
```

# Receiver interface
The [Send](#Send) and [SendFrom](#SendFrom) functions optionally send a
callback to a registered contract address, [as described above](#RegisterReceive).
The callback in both cases will include a message with the following format:

```json
{
  "receive": {
    "sender": "address of the sender",
    "from": "address of the owner of the funds",
    "amount": "funds that were sent as Uint128",
    "msg": "custom message, optional"
  }
}
```

The `sender` and `from` fields may be different if the `receive` message is
sent using the [SendFrom](#SendFrom) message. They will be the same when sent
by a `Send` call.

If an optional `msg` field is included in the message to the SNIP-20
contract, then it will be included in the `receive` message sent to the
Receiver contract.
