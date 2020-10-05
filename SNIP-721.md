# WIP ~ SNIP-721 Spec: Private, Non-Fungible Tokens

SNIP-721, (a.k.a. Secret-721) is a specification for privacy-preserving, non-fungible tokens based on CosmWasm on the Secret Network. The name and design is loosely based on Ethereum's [ERC-721](https://eips.ethereum.org/EIPS/eip-721) and a superset of CosmWasm's [CW721](https://github.com/CosmWasm/cosmwasm-plus/blob/master/packages/cw721/README.md). The additions to this spec are mostly for privacy features, and as such will strive to maintain compatability with CW721. 

The following specification is organized into multiple sections. A contract may only implement some of this functionality, but must implement the base.

## Scope

This document aims to establish standard interfaces that Secret-721 contract implementors will create, and that both wallet implementors & dependent contract creators will consume.
For this reason, the focus of this document is to merely give Secret-721 contract implementors the tools needed to create contracts that fully maintain privacy, but not to specify implmentation details.

## Notation

Notation in this document conforms to the standards set by [RFC-2119](http://microformats.org/wiki/rfc-2119)

## Terms

- __Message__ ~ This is an on-chain interface. It is triggered by sending a transaction, and receiving an on-chain response which is read by the client. Messages are authenticated by the blockchain and the secure enclave.
- __Query__ ~ This is an off-chain interface. Queries are done by returning data that a node has locally, and are not public. Query responses are returned immediately, and do not have to wait for blocks. In addition, queries cannot be authenticated using the standard interfaces. Any contract that wishes to strongly enforce query permissions must implement it themselves.
- __Cosmos Message Sender__ ~ The account found under the `sender` field in a Cosmos SDK message. Also the signer of the message.

## Padding

Secret-721 tokens may want to enforce constant length messages to avoid leaking data. To support this functionality, an optional _padding_ field may be sent with ANY of the messages in this spec. Contracts and Clients MUST ignore this field if sent, either in the request or response fields

## Requests

Requests SHOULD be sent as base64 encoded JSON. Future versions of Secret Network may add support for other formats as well, but at this time we recommend usage of JSON only. For this reason the parameter decriptions specifiy the JSON type which must be used. In addition, request parameters will include in parentheses a CosmWasm (or other) underlying type that this value must conform to. E.g. a recipient address is sent as a string, but must also be parsed to a bech32 address. 

## Responses

Unless specified otherwise, all message & query responses will be JSON encoded in the `data` field of the Cosmos response.
The reason for this is to make data leakage via message-length side-channels harder. In addition, since all keys will be encrypted, it is not possible to use the `log` events
for event triggering.

## Token IDs
Note that all token IDs are parsed as Uint128 (128 bit integers with JSON string representation).

# Base

## Messages

### Transfer 
Moves a token from the account that appears in the Cosmos message sender[1] field to the account in the recipient field. 

* Variation from CW721: It is NOT required to validate that the recipient is an address and not a contract. This command will work when trying to send funds to contract accounts as well.

##### Request

|Name      |Type             |Description                                                                                                 | optional |
|----------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|recipient | string          |  Accounts SHOULD be a valid bech32 address, but contracts may use a different naming scheme as well.        |          |
|id        | string (Uint128)|  The ID of the token to transfer                                                                           |          |

##### Response

```json
{
	"transfer": {
		"status": "success"
	}
}
```

### Send
Moves a token from the Cosmos message sender[1] account to the recipient account, with the ability to register a receiver interface which will be called _after_ the transfer is completed. Contract MAY be an address of a contract that registered the Receiver interface. Msg is binary encoded data that can be decoded into a contract-specific message by the receiver, which will be passed to the recipient, along with the token ID.
Even if the receiver function fails for any reason, the actual transfer of a token MUST NOT be reverted.

##### Request parameters

|Name      |Type             |Description                                                                                                 | optional |
|----------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|recipient | string          |  Accounts SHOULD be a valid bech32 address, but contracts may use a different naming scheme as well.        |          |
|id        | string (Uint128)|  The ID of the token to send                                                                               |          |
|msg       | string (base64) |                                                                                                            | yes      |

##### Response

```json
{
	"send": {
		"status": "success"
	}
}
```

### Burn
MUST remove amount tokens from the balance of Cosmos message sender[1] and MUST reduce the total supply by the same amount. 

##### Request
|Name      |Type             |Description                                                                                                 | optional |
|----------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|id        | string (Uint128)|  The ID of the token to burn                                                                                  |          |

##### Response

```json
{
	"burn": {
		"status": "success"
	}
}
```

### RegisterReceive
This message is used to tell the Secret-721 contract to call the _Receive_ function of the Cosmos message sender[1] after a successful _send_. 

In Secret Network this is used to pair a code hash with the contract address that must be called. This means that the Secret-721 MUST store the sent code_hash and use it when calling the _Receive_ function

TBD - is there any value in allowing an address other than msg.sender? i.e. If I wanted every time someone sends me tokens to trigger some _other_ contract.

##### Request
|Name         |Type             |Description                                                                                                 | optional |
|-------------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|code_hash    | string          |  A 32-byte hex encoded string, with the code hash of the receiver contract                                 |          |

##### Response

```json
{
	"register_receive": {
		"status": "success"
	}
}
```

### CreateViewingKey
This function _generates a new_ viewing key for the Cosmos message sender[1], which is used in ALL account-specific queries. This key is used to validate the identity of the caller, as in queries there is no way to cryptographically authenticate the caller.
	
The Viewing Key MUST be stored in such a way that lookup takes a significant amount to time to perform, in order to be resistent to brute-force attacks.
The viewing key MUST NOT control any functions which actively affect the balance of the user.
	
##### Request Parameters

|Name         |Type             |Description                                                                                                 | optional |
|-------------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|entropy      | string          |  A user-supplied string used for entropy for generation of the viewing key...<br>Secure implementation is left to the client, but it is recommended to use base-64 encoded random bytes and not predictable inputs.       |          |
	
##### Response
```json
{
	"create_viewing_key": {
		"api_key": "<string>"
	}
}
```

### SetViewingKey

Set a viewing key with a predefined value for Cosmos message sender[1], _without_ creating it. This is useful to manage multiple Secret-721 tokens using the same viewing key.

If a viewing key is already set, the contract MUST replace the current key. 
If a viewing key is not set, the contract MUST set the provided key as the viewing key.

It is NOT RECOMMENDED to use this function to create easy to remember passwords for users, but this is left up to implementors to enforce.

##### Request Parameters

|Name         |Type             |Description                                                                                                 | optional |
|-------------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|viewing_key  | string          |  A user-supplied string that will be used to authenticate the sender                                       |          |

##### Response
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
Authentication MUST be a resource intensive operation, that takes a significant[2] amount of time to compute. This is because such queries are open to offline brute-force attacks, which can be parallelized to scale linearly with the resources of a motivated attacker.
Authentication MUST perform the same computation even if the user does not have a viewing key set. 
Authentication response MUST be indistinguishable for both the case of a wrong viewing key and the case of a non-existant viewing key

### Collection - Authenticated
Returns a set of token IDs for the given address. Returns "0" if the address is unknown to the contract.

##### Request parameters

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

### TransferFrom

Transfer token(s) from a specified account, to another specified account.

##### Request

|Name         |Type             |Description                                                                                                 | optional |
|-------------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|owner        | string          |  Account to take a token __from__                                                                          |          |
|recipient    | string          |  Account to send a token __to__                                                                            |          |
|id           | string(Uint128) |  ID of the token to transfer                                                                               |          |

##### Response

```json
{
	"transfer_from": {
		"status": "success"
	}
}
```

### SendFrom
SendFrom is to Send, what TransferFrom is to Transfer. This allows a pre-approved account to not only transfer the token, 
but also send it to another address in order to trigger some action(s). Note SendFrom will set the Receive{sender} to be the env.sender 
(the account that triggered the transfer) rather than the owner account (the account the token is coming from).

##### Request

|Name         |Type             |Description                                                                                                 | optional |
|-------------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|owner        | string          |  Account to take a token __from__                                                                          |          |
|recipient    | string          |  Account to send a token __to__                                                                            |          |
|id           | string(Uint128) |  ID of the token to transfer                                                                               |          |
|msg          | string(base64)  |  Base64 encoded message, which the recipient will receive                                                  | yes      | 

##### Response

```json
{
	"send_from": {
		"status": "success"
	}
}
```

### BurnFrom
This works like TransferFrom, but it burns the token, instead of transferring it. 
This will remove the Secret NFT from the owner's wallet.

##### params
|Name         |Type             |Description                                                                                                 | optional |
|-------------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|owner        | string          |  Account to take a token __from__                                                                          |          |
|id           | string(Uint128) |  ID of the token to burn                                                                                   |          |

##### Response

```json
{
	"burn_from": {
		"status": "success"
	}
}
```

# Mintable
This allows another contract to mint a new token for a specified recipient. 
There is only one minter specified here, if you want more complex access management, 
please use a multi-sig or another contract as the minter address. You may handle updating the ACL there.

## Messages
### Mint
If the Cosmos message sender[1] is an approved minter, this will create a new token for their designated recipient.

##### Request
|Name         |Type             |Description                                                                                                 | optional |
|-------------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
|recipient    | string          |  Account to mint a token __to__                                                                            |          |
|id           | string(Uint128) |  ID of the token to mint                                                                                   |          |

##### Response

```json
{
	"mint": {
		"status": "success",
	}
}
```

## Queries
### Minter
Returns who can mint. Return type is MinterResponse {minter}.
##### Params
  None

##### Response

```json
{
	"minter": {
		"minter": "<address>",
	}
}
```
