# SNIP-721: Private, Non-Fungible Tokens

SNIP-721, (a.k.a. Secret-721) is a specification for privacy-preserving, non-fungible tokens based on CosmWasm on the Secret Network. The name and design was inspired by Ethereum's [ERC-721](https://eips.ethereum.org/EIPS/eip-721) and CosmWasm's [CW721](https://github.com/CosmWasm/cosmwasm-plus/blob/master/packages/cw721/README.md). Our differences are mostly for privacy features, and as such, we will strive to maintain compatability with CW721. 

The following specification is organized into multiple sections. A contract may only implement some of this functionality, but must implement the base.

## Scope

This document aims to establish standard interfaces that Secret-721 contract implementors will create, and that both wallet implementors & dependent contract creators will consume. For this reason, the focus of this document is to merely give Secret-721 contract implementors the tools needed to create contracts that fully maintain privacy, but not to specify implmentation details.

## Notation

Notation in this document conforms to the standards set by [RFC-2119](http://microformats.org/wiki/rfc-2119)

## Terms

- __Message__ ~ This is an on-chain interface. It is triggered by sending a transaction, and receiving an on-chain response which is read by the client. Messages are authenticated by the blockchain and the secure enclave.
- __Query__ ~ This is an off-chain interface. Queries are done by returning data that a node has locally, and are not public. Query responses are returned immediately, and do not have to wait for blocks. In addition, queries cannot be authenticated using the standard interfaces. Any contract that wishes to strongly enforce query permissions must implement it themselves.
- __Cosmos Message Sender__ ~ The account found under the `sender` field in a Cosmos SDK message.

## Padding

Secret-721 tokens may want to enforce constant length messages to avoid leaking data. To support this functionality, an optional _padding_ field may be sent with ANY of the messages in this spec. Contracts and Clients MUST ignore this field if sent, either in the request or response fields

## Requests

Requests SHOULD be sent as base64 encoded JSON. Future versions of Secret Network may add support for other formats as well, but at this time we recommend usage of JSON only. For this reason the parameter decriptions specifiy the JSON type which must be used. In addition, request parameters will include in parentheses a CosmWasm (or other) underlying type that this value must conform to. E.g. a recipient address is sent as a string, but must also be parsed to a bech32 address. 

## Responses

Unless specified otherwise, all message & query responses will be JSON encoded in the `data` field of the Cosmos response.
The reason for this is to make data leakage via message-length side-channels harder. In addition, since all keys will be encrypted, it is not possible to use the `log` events for event triggering.

## Token IDs
Note that all token IDs are parsed as Uint128 (128 bit integers with JSON string representation).

# Base

This handles ownership, transfers, and allowances. These must be consistently supported by all Secret-721 contracts. Note that all tokens must have an owner, as well as an ID. The ID is an arbitrary string, unique within the contract.

## Messages

### Transfer

Moves a Secret NFT from the account in the Cosmos message sender field to the account in the recipient field.

`TransferNft{recipient, token_id}` - 
This transfers ownership of the token to `recipient` account. This is 
designed to send to an address controlled by a private key and *does not* 
trigger any actions on the recipient if it is a contract.

Requires `token_id` to point to a valid token, and `env.message.sender` to be 
the owner of it, or have an allowance to transfer it. 

##### Request Parameters

|Name      |Type             |Description                                                                                                 | optional |
|----------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|recipient | string          |  Accounts SHOULD be a valid bech32 address, but contracts may use a different naming scheme as well.       |          |
|token_id  | string (Uint128)|  The ID of the token to transfer                                                                           |          |

##### Response Format

```json
{
	"transfer": {
		"status": "success"
	}
}
```

### Send

Moves a token from the Cosmos message sender account to the recipient account, with the ability to register a receiver interface which will be called _after_ the transfer is completed.

`SendNft{contract, token_id, msg}` - 
This transfers ownership of the token to `contract` account. `contract` 
MUST be an address controlled by a secret contract, which implements
the Receiver interface. The `msg` will be passed to the recipient 
contract, along with the token_id.

Requires `token_id` to point to a valid token, and `env.message.sender` to be 
the owner of it, or have an allowance to transfer it.

##### Request Parameters

|Name      |Type             |Description                                                                                                 | optional |
|----------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|contract  | string          |  Accounts SHOULD be a valid bech32 address, but contracts may use a different naming scheme as well.       |          |
|token_id  | string (Uint128)|  The ID of the token to send                                                                               |          |
|msg       | string (base64) |                                                                                                            | yes      |

##### Response Format

```json
{
	"send": {
		"status": "success"
	}
}
```

### Approve / Revoke

`Approve{spender, token_id, expires}` - Grants permission to `spender` to
transfer or send the given token. This can only be performed when
`env.message.sender` is the owner of the given `token_id` or an `operator`. 
There can multiple spender accounts per token, and they are cleared once
the token is transfered or sent.

`Revoke{spender, token_id}` - This revokes a previously granted permission
to transfer the given `token_id`. This can only be granted when
`env.message.sender` is the owner of the given `token_id` or an `operator`.

`ApproveAll{operator, expires}` - Grant `operator` permission to transfer or send
all tokens owned by `env.message.sender`. This approval is tied to the owner, not the
tokens and applies to any future token that the owner receives as well.

`RevokeAll{operator}` - Revoke a previous `ApproveAll` permission granted
to the given `operator`.

### Queries

`OwnerOf{token_id}` - Returns the owner of the given token,
as well as anyone with approval on this particular token.
If the token is unknown, returns an error. Return type is
`OwnerResponse{owner}`.

`ApprovedForAll{owner}` - List all operators that can access all of 
the owner's tokens. Return type is `ApprovedForAllResponse`

`NumTokens{}` - Total number of tokens issued

### Receiver

The counter-part to `SendNft` is `ReceiveNft`, which must be implemented by
any contract that wishes to manage CW721 tokens. This is generally *not*
implemented by any CW721 contract.

`ReceiveNft{sender, token_id, msg}` - This is designed to handle `SendNft`
messages. The address of the contract is stored in `env.message.sender`
so it cannot be faked. The contract should ensure the sender matches
the token contract it expects to handle, and not allow arbitrary addresses.

The `sender` is the original account requesting to move the token
and `msg` is a `Binary` data that can be decoded into a contract-specific
message. This can be empty if we have only one default action,
or it may be a `ReceiveMsg` variant to clarify the intention. For example,
if I send to an exchange, I can specify the price for the token.

---

### RegisterReceive
This message is used to tell the Secret-721 contract to call the _Receive_ function of the Cosmos message sender after a successful _send_. 

In Secret Network this is used to pair a code hash with the contract address that must be called. This means that the Secret-721 MUST store the sent code_hash and use it when calling the _Receive_ function

TBD - is there any value in allowing an address other than msg.sender? i.e. If I wanted every time someone sends me tokens to trigger some _other_ contract.

##### Request Parameters

|Name         |Type             |Description                                                                                                 | optional |
|-------------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|code_hash    | string          |  A 32-byte hex encoded string, with the code hash of the receiver contract                                 |          |

##### Response Format

```json
{
	"register_receive": {
		"status": "success"
	}
}
```

### CreateViewingKey
This function _generates a new_ viewing key for the Cosmos message sender, which is used in ALL account-specific queries. This key is used to validate the identity of the caller, as in queries there is no way to cryptographically authenticate the caller.
	
The Viewing Key MUST be stored in such a way that lookup takes a significant amount to time to perform, in order to be resistent to brute-force attacks.
The viewing key MUST NOT control any functions which actively affect the balance of the user.
	
##### Request Parameters

|Name         |Type             |Description                                                                                                 | optional |
|-------------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|entropy      | string          |  A user-supplied string used for entropy for generation of the viewing key...<br>Secure implementation is left to the client, but it is recommended to use base-64 encoded random bytes and not predictable inputs.       |          |
	
##### Response Format

```json
{
	"create_viewing_key": {
		"api_key": "<string>"
	}
}
```

### SetViewingKey

Set a viewing key with a predefined value for Cosmos message sender, _without_ creating it. This is useful to manage multiple Secret-721 tokens using the same viewing key.

If a viewing key is already set, the contract MUST replace the current key. 
If a viewing key is not set, the contract MUST set the provided key as the viewing key.

It is NOT RECOMMENDED to use this function to create easy to remember passwords for users, but this is left up to implementors to enforce.

##### Request Parameters

|Name         |Type             |Description                                                                                                 | optional |
|-------------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|viewing_key  | string          |  A user-supplied string that will be used to authenticate the sender                                       |          |

##### Response Format

```json
{
	"set_viewing_key": {
		"api_key": "<string>"
	}
}
```

## Queries

Queries are off-chain requests, that are not cryptographically validated. This means that contracts that wish to validate the caller of a query MUST implement some sort of authentication. Secret-721 uses an "API key" scheme, which validates a <viewing key, account> pair.

Authentication MUST happen on each query that reveals private account-specific information.
Authentication MUST be a resource intensive operation, that takes a significant amount of time to compute. This is because such queries are open to offline brute-force attacks, which can be parallelized to scale linearly with the resources of a motivated attacker.
Authentication MUST perform the same computation even if the user does not have a viewing key set. 
Authentication response MUST be indistinguishable for both the case of a wrong viewing key and the case of a non-existant viewing key

##### Request Parameters

|Name         |Type             |Description                                                                                                 | optional |
|-------------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|viewing_key  | string          |  A user-supplied string that will be used to authenticate the sender                                       |          |
| address     | string          |  Addresses SHOULD be a valid bech32 address, but contracts may use a different naming scheme as well.      |          |


### TransferHistory - Authenticated

|Name         |Type             |Description                                                                                                  | optional |
|-------------|-----------------|-------------------------------------------------------------------------------------------------------------|----------|
|viewing_key  | string          | A user-supplied string that will be used to authenticate the sender                                         |          |
| address     | string          | Addresses SHOULD be a valid bech32 address, but contracts may use a different naming scheme as well.        |          |
| page_size   | number          | Number of transactions to return, starting from the latest, i.e., n=1 returns only the latest transaction   |          |
| page        | number          | This defaults to 0. Specifying a positive number will skip `page * page_size` txs from the start.           | yes      |
 
## Metadata

### Queries
All queries below that return only public metadata use the same names as their CW721 counterparts in order to maintain universal compatibility.

### ContractInfo
Returns top-level metadata about the contract.  Namely, `name` and `symbol`.

##### Request parameters
None

### NftInfo
Returns the public metadata about one particular token.  The return value is based on *ERC-721 Metadata JSON Schema*, but directly from the contract, not as a Uri. Only the image link is a Uri.

##### Request Parameters

|Name         |Type             |Description                                                                                                 | optional |
|-------------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|token_id  | string (Uint128)|  The ID of the token whose metadata should be displayed                                                         |          |

### AllNftInfo
Returns the result of both `NftInfo` and `OwnerOf` as one query as an optimization for clients needing both.

##### Request Parameters

|Name         |Type             |Description                                                                                                 | optional |
|-------------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|token_id  | string (Uint128)|  The ID of the token whose owner and metadata should be displayed                                              |          |

### PrivateNftInfo
Returns the public and private metadata about one particular token.  The return value is based on *ERC-721 Metadata JSON Schema*, but directly from the contract, not as a Uri. Only the image link is a Uri.<br/>
If the supplied viewing key is not correct, or if the NFT owner has not set a viewing key, the response should be identical to the response of `NftInfo`.<br/>
Authentication MUST be a resource intensive operation, that takes a significant amount of time to compute. This is because such queries are open to offline brute-force attacks, which can be parallelized to scale linearly with the resources of a motivated attacker.  In addition, authentication MUST perform the same computation even if the user does not have a viewing key set. 

TBD - Will use cases only require a single key-value pair of "string" type added to the *ERC-721 Metadata JSON Schema* base for the private metadata, or will it need to be an "object" type with multiple key-value pairs?

##### Request Parameters

|Name         |Type             |Description                                                                                                 | optional |
|-------------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|token_id  | string (Uint128)|  The ID of the token whose metadata should be displayed                                                    |          |
|viewing_key  | string          | API key used to authenticate access to private metadata                                                        |          |

## Enumerable

### Queries

Pagination is acheived via `start_after` and `limit`. Limit is a request
set by the client, if unset, the contract will automatically set it to
`DefaultLimit` (suggested 10). If set, it will be used up to a `MaxLimit`
value (suggested 30). Contracts can define other `DefaultLimit` and `MaxLimit`
values without violating the SNIP-721 spec, and clients should not rely on
any particular values.

If `start_after` is unset, the query returns the first results, ordered by
lexogaphically by `token_id`. If `start_after` is set, then it returns the
first `limit` tokens *after* the given one. This allows straight-forward 
pagination by taking the last result returned (a `token_id`) and using it
as the `start_after` value in a future query. 

`Tokens{owner, start_after, limit}` - List all token_ids that belong to a given owner.
Return type is `TokensResponse{tokens: Vec<token_id>}`.

`AllTokens{start_after, limit}` - Requires pagination. Lists all token_ids controlled by 
the contract.
