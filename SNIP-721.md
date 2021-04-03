# SNIP-721: Private, Non-Fungible Tokens
SNIP-721 is a specification for private non-fungible tokens based on CosmWasm on the Secret Network. The name and design is loosely based on [ERC-721](https://eips.ethereum.org/EIPS/eip-721), and is a superset of CosmWasm's [CW-721](https://github.com/CosmWasm/cosmwasm-plus/blob/master/packages/cw721/README.md). While this specification is CW-721 compliant, because CW-721 is not capable of privacy, a number of the CW-721-compliant functions may not return all the information a CW-721 implementation would.  For example, the [OwnerOf](#ownerof) query must not display the approvals for a token unless the token owner has supplied his address and viewing key.  In order to strive for CW-721 compliance, a number of queries that require authentication use optional parameters that the CW-721 counterpart does not have.  If the optional authentication parameters are not supplied, the responses must only display information that the token owner has made public.

The [SNIP-721 reference implementation](https://github.com/baedrik/snip721-reference-impl) may be used to create SNIP-721-compliant token contracts either as-is or as a base upon which to build additional application-specific functionality.

This specification is split into multiple sections, a contract may only implement some of this functionality, but must implement the base functionality.

## Scope
This document aims to set standard interfaces that SNIP-721 contract implementors will create, and that both wallet implementors & dependent contract creators will consume.  For this reason, the focus of this document is to merely give SNIP-721 contract implementors the tools needed to create contracts that fully maintain privacy but not to specify implementation details.  That said, this document may, at times, mention implementation details of the [SNIP-721 reference implementation](https://github.com/baedrik/snip721-reference-impl) as non-base functionality that developers might choose to mirror.

## Terms
- __Message__ - This is an on-chain interface. It is triggered by sending a transaction, and receiving an on-chain response which is read by the client. Messages are authenticated both by the blockchain, and by the secret enclave.
- __Query__ - This is an off-chain interface. Queries are done by returning data that a node has locally, and are not public. Query responses are returned immediately, and do not have to wait for blocks.
- __Cosmos Message Sender__ - The account that is found under the `sender` field in a standard Cosmos SDK message. This is also the signer of the message.

## Memo
Users may want to include private memos with transactions.  While it is possible to include a memo with the Cosmos message, that message is publicly viewable.  Therefore, to enable private memos, SNIP-721 token contracts must allow an optional `memo` field in any message that generates a mint, burn, or transfer transaction.

## Padding
Users may want to enforce constant length messages to avoid leaking data. To support this functionality, SNIP-721 token contracts must support the option to include a `padding` field in every message. This optional `padding` field may be sent with any of the messages in this spec. Contracts must ignore this field if sent.

## Requests
Requests should be sent as base64 encoded JSON. Future versions of Secret Network may add support for other formats as well, but at this time we recommend usage of JSON only. For this reason the parameter descriptions specify the JSON type which must be used. In addition, request parameters will include in parentheses a CosmWasm (or other) underlying type that this value must conform
to. E.g. a recipient address is sent as a string, but must also be parsed to a bech32 address.

## Queries
Queries are off-chain requests that are not cryptographically validated. This means that contracts that wish to validate the caller of a query must implement some sort of authentication. SNIP-721 uses an "API key" scheme, which validates a `(viewing key, account)` pair.

Authentication must happen on each query that reveals private account-specific information. Authentication must be a resource intensive operation that takes a significant amount of time to compute.  This is because such queries are open to offline brute-force attacks, which can be parallelized to scale linearly with the resources of a motivated attacker.  Authentication must perform the same computation even if the user does not have a viewing key set.  The authentication response must be indistinguishable for both the case of a wrong viewing key and the case of a non-existent viewing key.

<a name="queryblockinfo"></a>One should be aware that the current blockheight and time is not available to a query on Secret Network at this moment, but there are plans to make the BlockInfo available to queries in a future hardfork.  To get around this limitation, the SNIP-721 contract may choose to store the BlockInfo every time a message is executed, in order to use the blockheight and time of the last message execution when checking the expiration of an approval during a query.  Therefore it is possible that a whitelisted address may be able to view the owner or metadata of a token past its approval expiration if no one executed any contract message since before the expiration.

## Responses
Unless otherwise specified, all message & query responses will be JSON encoded in the `data` field of the Cosmos response, rather than in the `logs`.  This is meant to reduce the potential for data-leakage through side-channel attacks. In addition, since all keys will be encrypted, it is not possible to use the `log` events for event triggering.

### Success status
Some of the messages detailed in this document contain a `status` field.  This field must hold one of two values: "success" or "failure".

While errors during execution of contract functions should usually result in a proper and detailed error response, The "failure" status is reserved for cases where a contract might choose to obfuscate the exact cause of failure, or otherwise indicate that while nothing failed to happen, the operation itself could not be completed for some valid reason.

# Base

This handles ownership, transfers, approvals, and metadata. These messages and queries must be consistently supported by all SNIP-721 contracts; however, all [metadata](#metadata) response fields are optional to allow for SNIP-721 contracts that choose not to implement metadata. Note that all tokens must have an owner as well as an ID. The ID is an arbitrary string, unique within the contract.

## Messages

### TransferNft
TransferNft is used to transfer ownership of the token to the `recipient` address.  This requires a valid `token_id` and the message sender must either be the owner or an address with valid transfer approval.  If the token is transferred to a new owner, its single-token approvals must be cleared.

##### Request
```
{
	"transfer_nft": {
		"recipient": "address_receiving_the_token",
		"token_id": "ID_of_the_token_being_transferred",
		"memo": "optional_memo_for_the_transfer_tx",
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name      | Type               | Description                                                                                                                        | Optional | Value If Omitted |
|-----------|--------------------|------------------------------------------------------------------------------------------------------------------------------------|----------|------------------|
| recipient | string (HumanAddr) | Address receiving the token                                                                                                        | no       |                  |
| token_id  | string             | Identifier of the token to be transferred                                                                                          | no       |                  |
| memo      | string             | `memo` for the transfer transaction that is only viewable by addresses involved in the transfer (recipient, sender, previous owner)| yes      | nothing          |
| padding   | string             | An ignored string that can be used to maintain constant message length                                                             | yes      | nothing          |

##### Response
```
{
	"transfer_nft": {
		"status": "success"
	}
}
```

### <a name="sendnft"></a>SendNft
SendNft is used to transfer ownership of the token to the `contract` address, and then call the recipient's BatchReceiveNft (or ReceiveNft, [see below](#receiver)) if the recipient contract has registered its receiver interface with the NFT contract.  While SendNft keeps the `contract` field name in order to maintain CW-721 compliance, Secret Network does not have the same limitations as Cosmos, and it is possible to use SendNft to transfer token ownership to a personal address (not a contract) or to a contract that does not implement any [Receiver Interface](#receiver).

SendNft requires a valid `token_id` and the message sender must either be the owner or an address with valid transfer approval.  If the token is transferred to a new owner, its single-token approvals must be cleared.  If the BatchReceiveNft (or ReceiveNft) callback fails, the entire transaction must be reverted (even the transfer must not take place).

##### Request
```
{
	"send_nft": {
		"contract": "address_receiving_the_token",
		"token_id": "ID_of_the_token_being_transferred",
		"msg": "optional_base64_encoded_Binary_message_sent_with_the_BatchReceiveNft_callback",
		"memo": "optional_memo_for_the_transfer_tx",
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name     | Type                           | Description                                                                                                                | Optional | Value If Omitted |
|----------|--------------------------------|----------------------------------------------------------------------------------------------------------------------------|----------|------------------|
| contract | string (HumanAddr)             | Address receiving the token                                                                                                | no       |                  |
| token_id | string                         | Identifier of the token to be transferred                                                                                  | no       |                  |
| msg      | string (base64 encoded Binary) | `msg` included when calling the recipient contract's BatchReceiveNft (or ReceiveNft)                                       | yes      | nothing          |
| memo     | string                         | `memo` for the transfer tx that is only viewable by addresses involved in the transfer (recipient, sender, previous owner) | yes      | nothing          |
| padding  | string                         | An ignored string that can be used to maintain constant message length                                                     | yes      | nothing          |

##### Response
```
{
	"send_nft": {
		"status": "success"
	}
}
```

### Approve
Approve is used to grant an address permission to transfer a single token.  This can only be performed by the token's owner or, in compliance with CW-721, an address that has inventory-wide approval to transfer the owner's tokens.  Approve is provided to maintain compliance with CW-721, but the owner can use [SetWhitelistedApproval](#setwhitelisted) to accomplish the same thing if specifying a `token_id` and `approve_token` [AccessLevel](#accesslevel) for `transfer`.

##### Request
```
{
	"approve": {
		"spender": "address_being_granted_approval_to_transfer_the_specified_token",
		"token_id": "ID_of_the_token_that_can_now_be_transferred_by_the_spender",
		"expires": "never" | {"at_height": 999999} | {"at_time":999999},
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name                  | Type                                     | Description                                                                                          | Optional | Value If Omitted |
|-----------------------|------------------------------------------|------------------------------------------------------------------------------------------------------|----------|------------------|
| spender               | string (HumanAddr)                       | Address being granted approval to transfer the token                                                 | no       |                  |
| token_id              | string                                   | ID of the token that the spender can now transfer                                                    | no       |                  |
| expires               | [Expiration (see below)](#expiration)    | The expiration of this token transfer approval.  Can be a blockheight, time, or never                | yes      | "never"          |
| padding               | string                                   | An ignored string that can be used to maintain constant message length                               | yes      | nothing          |

##### Response
```
{
	"approve": {
		"status": "success"
	}
}
```

#### <a name="expiration"></a>Expiration
The Expiration object is used to set an expiration for any approvals granted in the message.  Expiration can be set to a specified blockheight, a time in seconds since epoch 01/01/1970, or "never".  Values for blockheight and time are specified as a u64.  If no expiration is given, it must default to "never".

Also, because the current blockheight and time will not be available to queries until a future hardfork makes that possible, please [see above](#queryblockinfo) regarding an imprecise, and possibly delayed way to enforce expirations on queries in the meantime.  This imprecise method must only be applied when checking an expiration during a query.  When checking an expiration during a message, the blockheight and time are available and exact expiration must be enforced.

* `"never"` - the approval will never expire
* `{"at_time": 1700000000}` - the approval will expire 1700000000 seconds after 01/01/1970 (time value is u64)
* `{"at_height": 3000000}` - the approval will expire at blockheight 3000000 (height value is u64)

### Revoke
Revoke is used to revoke from an address the permission to transfer this single token.  This can only be performed by the token's owner or, in compliance with CW-721, an address that has inventory-wide approval to transfer the owner's tokens (referred to as an operator later). However, one operator may not revoke transfer permission of even one single token away from another operator.  Revoke is provided to maintain compliance with CW-721, but the owner can use [SetWhitelistedApproval](#setwhitelisted) to accomplish the same thing if specifying a `token_id` and `revoke_token` [AccessLevel](#accesslevel) for `transfer`.

##### Request
```
{
	"revoke": {
		"spender": "address_being_revoked_approval_to_transfer_the_specified_token",
		"token_id": "ID_of_the_token_that_can_no_longer_be_transferred_by_the_spender",
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name                  | Type                    | Description                                                                                          | Optional | Value If Omitted |
|-----------------------|-------------------------|------------------------------------------------------------------------------------------------------|----------|------------------|
| spender               | string (HumanAddr)      | Address no longer permitted to transfer the token                                                    | no       |                  |
| token_id              | string                  | ID of the token that the spender can no longer transfer                                              | no       |                  |
| padding               | string                  | An ignored string that can be used to maintain constant message length                               | yes      | nothing          |

##### Response
```
{
	"revoke": {
		"status": "success"
	}
}
```

### ApproveAll
ApproveAll is used to grant an address permission to transfer all the tokens in the message sender's inventory.  This must include the ability to transfer any tokens the sender acquires after granting this inventory-wide approval.  This also gives the address the ability to grant another address the approval to transfer a single token.  ApproveAll is provided to maintain compliance with CW-721, but the message sender can use [SetWhitelistedApproval](#setwhitelisted) to accomplish the same thing by using `all` [AccessLevel](#accesslevel) for `transfer`.

##### Request
```
{
	"approve_all": {
		"operator": "address_being_granted_inventory-wide_approval_to_transfer_tokens",
		"expires": "never" | {"at_height": 999999} | {"at_time":999999},
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name                  | Type                                     | Description                                                                                          | Optional | Value If Omitted |
|-----------------------|------------------------------------------|------------------------------------------------------------------------------------------------------|----------|------------------|
| operator              | string (HumanAddr)                       | Address being granted approval to transfer all of the message sender's tokens                        | no       |                  |
| expires               | [Expiration (see above)](#expiration)    | The expiration of this inventory-wide transfer approval.  Can be a blockheight, time, or never       | yes      | "never"          |
| padding               | string                                   | An ignored string that can be used to maintain constant message length                               | yes      | nothing          |

##### Response
```
{
	"approve_all": {
		"status": "success"
	}
}
```

### RevokeAll
RevokeAll is used to revoke all transfer approvals granted to an address.  RevokeAll is provided to maintain compliance with CW-721, but the message sender can use [SetWhitelistedApproval](#setwhitelisted) to accomplish the same thing by using `none` [AccessLevel](#accesslevel) for `transfer`.

##### Request
```
{
	"revoke_all": {
		"operator": "address_being_revoked_all_approvals_to_transfer_tokens",
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name                  | Type                    | Description                                                                                          | Optional | Value If Omitted |
|-----------------------|-------------------------|------------------------------------------------------------------------------------------------------|----------|------------------|
| operator              | string (HumanAddr)      | Address being revoked all approvals to transfer the message sender's tokens                          | no       |                  |
| padding               | string                  | An ignored string that can be used to maintain constant message length                               | yes      | nothing          |

##### Response
```
{
	"revoke_all": {
		"status": "success"
	}
}
```

### <a name="setwhitelisted"></a>SetWhitelistedApproval
The owner of a token can use SetWhitelistedApproval to grant an address permission to view ownership, view private metadata, and/or to transfer a single token or every token in the owner's inventory.  SetWhitelistedApproval can also be used to revoke any approval previously granted to the address.

##### Request
```
{
	"set_whitelisted_approval": {
		"address": "address_being_granted_or_revoked_approval",
		"token_id": "optional_ID_of_the_token_to_grant_or_revoke_approval_on",
		"view_owner": "approve_token" | "all" | "revoke_token" | "none",
		"view_private_metadata": "approve_token" | "all" | "revoke_token" | "none",
		"transfer": "approve_token" | "all" | "revoke_token" | "none",
		"expires": "never" | {"at_height": 999999} | {"at_time":999999},
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name                  | Type                                       | Description                                                                                        | Optional | Value If Omitted |
|-----------------------|--------------------------------------------|----------------------------------------------------------------------------------------------------|----------|------------------|
| address               | string (HumanAddr)                         | Address to grant or revoke approval to/from                                                        | no       |                  |
| token_id              | string                                     | If supplying either `approve_token` or `revoke_token` access, the token whose privacy is being set | yes      | nothing          |
| view_owner            | [AccessLevel (see below)](#accesslevel)    | Grant or revoke the address' permission to view the ownership of a token/inventory                 | yes      | nothing          |
| view_private_metadata | [AccessLevel (see below)](#accesslevel)    | Grant or revoke the address' permission to view the private metadata of a token/inventory          | yes      | nothing          |
| transfer              | [AccessLevel (see below)](#accesslevel)    | Grant or revoke the address' permission to transfer a token/inventory                              | yes      | nothing          |
| expires               | [Expiration (see above)](#expiration)      | The expiration of any approval granted in this message.  Can be a blockheight, time, or never      | yes      | "never"          |
| padding               | string                                     | An ignored string that can be used to maintain constant message length                             | yes      | nothing          |

##### Response
```
{
	"set_whitelisted_approval": {
		"status": "success"
	}
}
```
#### <a name="accesslevel"></a>AccessLevel
AccessLevel determines the type of access being granted or revoked to the specified address in a [SetWhitelistedApproval](#setwhitelisted) message or to everyone in a [SetGlobalApproval](#setglobal) message.  Inventory-wide approval and token-specific approval are mutually exclusive levels of access.  The levels are:
* `"approve_token"` - grant approval only on the token specified in the message
* `"revoke_token"` - revoke a previous approval on the specified token
* `"all"` - grant approval for all tokens in the message signer's inventory.  This approval must also apply to any tokens the signer acquires after granting `all` approval
* `"none"` - revoke any approval (both token and inventory-wide) previously granted to the specified address (or for everyone if using SetGlobalApproval)

### <a name="registerreceive"></a>RegisterReceiveNft
A contract will use RegisterReceiveNft to notify the SNIP-721 contract that it implements ReceiveNft and possibly also BatchReceiveNft [(see below)](#receiver).  This enables the SNIP-721 contract to call the registered contract whenever it is Sent a token (or tokens).  In order to comply with CW-721, ReceiveNft only informs the recipient contract that it has been sent a single token, and it only informs the recipient contract who the token's previous owner was, not who sent the token (which may be different addresses) despite calling the previous owner `sender` ([see below](#cwsender)).  BatchReceiveNft, on the other hand, can be used to inform a contract that it was sent multiple tokens, and notifies the recipient of both, the token's previous owner and the sender.

##### Request
```
{
	"register_receive_nft": {
		"code_hash": "code_hash_of_the_contract_implementing_a_receiver_interface",
		"also_implements_batch_receive_nft": true | false,
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name                              | Type   | Description                                                                                                                | Optional | Value If Omitted |
|-----------------------------------|--------|----------------------------------------------------------------------------------------------------------------------------|----------|------------------|
| code_hash                         | string | A 32-byte hex encoded string, with the code hash of the message sender, which is a contract that implements a receiver     | no       |                  |
| also_implements_batch_receive_nft | bool   | true if the message sender contract also implements BatchReceiveNft so it can be informed that it was sent a list of tokens| yes      | false            |
| padding                           | string | An ignored string that can be used to maintain constant message length                                                     | yes      | nothing          |

##### Response
```
{
	"register_receive_nft": {
		"status": "success"
	}
}
```

### CreateViewingKey
CreateViewingKey generates a new viewing key for the Cosmos message sender, which is used to authenticate account-specific queries, because queries in Cosmos have no way to cryptographically authenticate the querier's identity.

The Viewing Key must be implemented in such a way that validation takes a significant amount to time to perform, in order to be resistant to brute-force attacks. The viewing key must only be used to authenticate queries, as messages cryptographically authenticate the sender.

The `entropy` field of the request should be a client supplied string used for entropy for generation of the viewing key. Secure implementation is left to the contract developer, but it is recommended to use base-64 encoded random bytes and not predictable inputs.

##### Request
```
{
	"create_viewing_key": {
		"entropy": "string_used_as_part_of_the_entropy_supplied_to_the_rng",
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name      | Type   | Description                                                                                                        | Optional | Value If Omitted |
|-----------|--------|--------------------------------------------------------------------------------------------------------------------|----------|------------------|
| entropy   | string | String used as part of the entropy supplied to the rng that generates the random viewing key                       | no       |                  |
| padding   | string | An ignored string that can be used to maintain constant message length                                             | yes      | nothing          |

##### Response
```
{
	"viewing_key": {
		"key": "the_created_viewing_key"
	}
}
```

### SetViewingKey
SetViewingKey is used to set the viewing key to a predefined string.  It must replace any key that currently exists.  It would be best for users to call CreateViewingKey to ensure a strong key, but this function is provided so that contracts can also utilize viewing keys.

##### Request
```
{
	"set_viewing_key": {
		"key": "the_new_viewing_key",
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name      | Type   | Description                                                                                                        | Optional | Value If Omitted |
|-----------|--------|--------------------------------------------------------------------------------------------------------------------|----------|------------------|
| key       | string | The new viewing key for the message sender                                                                         | no       |                  |
| padding   | string | An ignored string that can be used to maintain constant message length                                             | yes      | nothing          |

##### Response
```
{
	"viewing_key": {
		"key": "the_message_sender's_viewing_key"
	}
}
```

## Queries

### ContractInfo
ContractInfo returns the contract's name and symbol.  This query is not authenticated.

##### Request
```
{
	"contract_info": {}
}
```
##### Response
```
{
	"contract_info": {
		"name": "contract_name",
		"symbol": "contract_symbol"
	}
}
```

### NumTokens
NumTokens returns the number of tokens controlled by the contract.  If the contract's token supply is private, the SNIP-721 contract may choose to only allow an authenticated minter's address to perform this query.

##### Request
```
{
	"num_tokens": {
		"viewer": {
			"address": "address_of_the_querier_if_supplying_optional_ViewerInfo",
			"viewing_key": "viewer's_key_if_supplying_optional_ViewerInfo"
		}
	}
}
```
| Name   | Type                                  | Description                                                         | Optional | Value If Omitted |
|--------|---------------------------------------|---------------------------------------------------------------------|----------|------------------|
| viewer | [ViewerInfo (see below)](#viewerinfo) | The address and viewing key performing this query                   | yes      | nothing          |

##### Response
```
{
	"num_tokens": {
		"count": 99999
	}
}
```
| Name    | Type         | Description                                  | Optional | 
|---------|--------------|----------------------------------------------|----------|
| count   | number (u32) | Number of tokens controlled by this contract | no       |

#### <a name="viewerinfo"></a>ViewerInfo
The ViewerInfo object provides the address and viewing key of the querier.  It is optionally provided in queries where public responses and address-specific responses will differ.
```
{
	"address": "address_of_the_querier_if_supplying_optional_ViewerInfo",
	"viewing_key": "viewer's_key_if_supplying_optional_ViewerInfo"
}
```
| Name        | Type               | Description                                                                                                           | Optional | Value If Omitted |
|-------------|--------------------|-----------------------------------------------------------------------------------------------------------------------|----------|------------------|
| address     | string (HumanAddr) | Address performing the query                                                                                          | no       |                  |
| viewing_key | string             | The querying address' viewing key                                                                                     | no       |                  |

### AllTokens
AllTokens returns an optionally paginated, lexicographically ordered list of all the token IDs controlled by the contract.  If the contract's token supply is private, the SNIP-721 contract may choose to only allow an authenticated minter's address to perform this query.  When paginating, supply the last token ID received in a response as the `start_after` string of the next query to continue listing where the previous query stopped.

##### Request
```
{
	"all_tokens": {
		"viewer": {
			"address": "address_of_the_querier_if_supplying_optional_ViewerInfo",
			"viewing_key": "viewer's_key_if_supplying_optional_ViewerInfo"
		},
		"start_after": "optional_string_where_results_will_only_be_token_IDs_that_come_later_in_lexicographical_order",
		"limit": 10
	}
}
```
| Name        | Type                                  | Description                                                                              | Optional | Value If Omitted       |
|-------------|---------------------------------------|------------------------------------------------------------------------------------------|----------|------------------------|
| viewer      | [ViewerInfo (see above)](#viewerinfo) | The address and viewing key performing this query                                        | yes      | nothing                |
| start_after | string                                | Results must only list token IDs that come after this string in lexicographical order    | yes      | nothing                |
| limit       | number (u32)                          | Number of token IDs to return                                                            | yes      | developer's discretion |

##### Response
```
{
	"token_list": {
		"tokens": [
			"list", "of", "token", "IDs", "controlled", "by", "the", "contract", "..."
		]
	}
}
```
| Name    | Type            | Description                                                          | Optional | 
|---------|-----------------|----------------------------------------------------------------------|----------|
| tokens  | array of string | A list of token IDs controlled by this contract                      | no       |

### <a name="ownerof"></a>OwnerOf
OwnerOf returns the owner of the specified token if the querier is the owner or has been granted permission to view the owner.  If the querier is the owner, OwnerOf must also display all the addresses that have been given transfer permission.  The transfer approval list is provided as part of CW-721 compliance; however, the token owner is advised to use [NftDossier (see below)](#nftdossier) for a more complete list that includes view_owner and view_private_metadata approvals (which CW-721 is not capable of keeping private).  If no [viewer](#viewerinfo) is provided, OwnerOf must only display the owner if ownership is public for this token.

##### Request
```
{
	"owner_of": {
		"token_id": "ID_of_the_token_being_queried",
		"viewer": {
			"address": "address_of_the_querier_if_supplying_optional_ViewerInfo",
			"viewing_key": "viewer's_key_if_supplying_optional_ViewerInfo"
		},
		"include_expired": true | false
	}
}
```
| Name            | Type                                  | Description                                                           | Optional | Value If Omitted |
|-----------------|---------------------------------------|-----------------------------------------------------------------------|----------|------------------|
| token_id        | string                                | ID of the token being queried                                         | no       |                  |
| viewer          | [ViewerInfo (see above)](#viewerinfo) | The address and viewing key performing this query                     | yes      | nothing          |
| include_expired | bool                                  | True if expired transfer approvals should be included in the response | yes      | false            |

##### Response
```
{
	"owner_of": {
		"owner": "address_of_the_token_owner",
		"approvals": [
			{
				"spender": "address_with_transfer_approval",
				"expires": "never" | {"at_height": 999999} | {"at_time":999999}
			},
			{
				"...": "..."
			}
		]
	}
}
```
| Name      | Type                                                 | Description                                              | Optional | 
|-----------|------------------------------------------------------|----------------------------------------------------------|----------|
| owner     | string (HumanAddr)                                   | Address of the token's owner                             | no       |
| approvals | array of [Cw721Approval (see below)](#cw721approval) | List of approvals to transfer this token                 | no       |

#### <a name="cw721approval"></a>Cw721Approval
The Cw721Approval object is used to display CW-721-style approvals which are limited to only permission to transfer, as CW-721 does not enable ownership or metadata privacy.
```
{
	"spender": "address_with_transfer_approval",
	"expires": "never" | {"at_height": 999999} | {"at_time":999999}
}
```
| Name    | Type                                  | Description                                                                     | Optional | 
|---------|---------------------------------------|---------------------------------------------------------------------------------|----------|
| spender | string (HumanAddr)                    | Address whitelisted to transfer a token                                         | no       |
| expires | [Expiration (see above)](#expiration) | The expiration of this transfer approval.  Can be a blockheight, time, or never | no       |

### <a name="nftinfo"></a>NftInfo
NftInfo returns the public [metadata](#metadata) of a token.  All metadata fields are optional to allow for SNIP-721 contracts that choose not to implement metadata.  It follows CW-721 specification, which is based on ERC-721 Metadata JSON Schema.

##### Request
```
{
	"nft_info": {
		"token_id": "ID_of_the_token_being_queried"
	}
}
```
| Name            | Type                                  | Description                                                           | Optional | Value If Omitted |
|-----------------|---------------------------------------|-----------------------------------------------------------------------|----------|------------------|
| token_id        | string                                | ID of the token being queried                                         | no       |                  |

##### Response
```
{
	"nft_info": {
		"name": "optional_name_of_the_token",
		"description": "optional_description",
		"image": "optional_uri_containing_an_image_or_additional_metadata"
	}
}
```
| Name        | Type   | Description                                              | Optional | 
|-------------|--------|----------------------------------------------------------|----------|
| name        | string | Name of the token                                        | yes      |
| description | string | Token description                                        | yes      |
| image       | string | Uri to an image or additional metadata                   | yes      |

#### <a name="metadata"></a>Metadata
Metadata for a token that follows CW-721 metadata specification, which is based on ERC721 Metadata JSON Schema.
```
{
	"name": "optional_name",
	"description": "optional_text_description",
	"image": "optional_uri_pointing_to_an_image_or_additional_off-chain_metadata"
}
```
| Name        | Type   | Description                                                           | Optional | Value If Omitted     |
|-------------|--------|-----------------------------------------------------------------------|----------|----------------------|
| name        | string | String that can be used to identify an asset's name                   | yes      | nothing              |
| description | string | String that can be used to describe an asset                          | yes      | nothing              |
| image       | string | String that can hold a link to additional off-chain metadata or image | yes      | nothing              |

### AllNftInfo
AllNftInfo displays the result of both [OwnerOf](#ownerof) and [NftInfo](#nftinfo) in a single query.  This is provided for CW-721 compliance, but for more complete information about a token, use [NftDossier](#nftdossier), which will include private metadata and view_owner and view_private_metadata approvals if the querier is permitted to view this information.

##### Request
```
{
	"all_nft_info": {
		"token_id": "ID_of_the_token_being_queried",
		"viewer": {
			"address": "address_of_the_querier_if_supplying_optional_ViewerInfo",
			"viewing_key": "viewer's_key_if_supplying_optional_ViewerInfo"
		},
		"include_expired": true | false
	}
}
```
| Name            | Type                                  | Description                                                           | Optional | Value If Omitted |
|-----------------|---------------------------------------|-----------------------------------------------------------------------|----------|------------------|
| token_id        | string                                | ID of the token being queried                                         | no       |                  |
| viewer          | [ViewerInfo (see above)](#viewerinfo) | The address and viewing key performing this query                     | yes      | nothing          |
| include_expired | bool                                  | True if expired transfer approvals should be included in the response | yes      | false            |

##### Response
```
{
	"all_nft_info": {
		"access": {
			"owner": "address_of_the_token_owner",
			"approvals": [
				{
					"spender": "address_with_transfer_approval",
					"expires": "never" | {"at_height": 999999} | {"at_time":999999}
				},
				{
					"...": "..."
				}
			]
		},
		"info": {
			"name": "optional_name_of_the_token",
			"description": "optional_description",
			"image": "optional_uri_containing_an_image_or_additional_metadata"
		}
	}
}
```
| Name        | Type                                              | Description                                                                 | Optional | 
|-------------|---------------------------------------------------|-----------------------------------------------------------------------------|----------|
| access      | [Cw721OwnerOfResponse (see below)](#cw721ownerof) | The token's owner and its transfer approvals if permitted to view this info | no       |
| info        | [Metadata (see above)](#metadata)                 | The token's public metadata                                                 | yes      |

#### <a name="cw721ownerof"></a>Cw721OwnerOfResponse
The Cw721OwnerOfResponse object is used to display a token's owner if the querier has view_owner permission, and the token's transfer approvals if the querier is the token's owner.
```
{
	"owner": "address_of_the_token_owner",
	"approvals": [
		{
			"spender": "address_with_transfer_approval",
			"expires": "never" | {"at_height": 999999} | {"at_time":999999}
		},
		{
			"...": "..."
		}
	]
}
```
| Name      | Type                                                 | Description                                              | Optional | 
|-----------|------------------------------------------------------|----------------------------------------------------------|----------|
| owner     | string (HumanAddr)                                   | Address of the token's owner                             | yes      |
| approvals | array of [Cw721Approval (see above)](#cw721approval) | List of approvals to transfer this token                 | no       |

### PrivateMetadata
PrivateMetadata returns the private [metadata](#metadata) of a token if the querier is permitted to view it.  All metadata fields are optional to allow for SNIP-721 contracts that choose not to implement private metadata.  It follows CW-721 metadata specification, which is based on ERC-721 Metadata JSON Schema.  If no [viewer](#viewerinfo) is provided, PrivateMetadata must only display the private metadata if the private metadata is public for this token.

##### Request
```
{
	"private_metadata": {
		"token_id": "ID_of_the_token_being_queried",
		"viewer": {
			"address": "address_of_the_querier_if_supplying_optional_ViewerInfo",
			"viewing_key": "viewer's_key_if_supplying_optional_ViewerInfo"
		},
	}
}
```
| Name            | Type                                  | Description                                                           | Optional | Value If Omitted |
|-----------------|---------------------------------------|-----------------------------------------------------------------------|----------|------------------|
| token_id        | string                                | ID of the token being queried                                         | no       |                  |
| viewer          | [ViewerInfo (see above)](#viewerinfo) | The address and viewing key performing this query                     | yes      | nothing          |

##### Response
```
{
	"private_metadata": {
		"name": "optional_private_name_of_the_token",
		"description": "optional_private_description",
		"image": "optional_private_uri_containing_an_image_or_additional_metadata"
	}
}
```
| Name        | Type   | Description                                                      | Optional | 
|-------------|--------|------------------------------------------------------------------|----------|
| name        | string | Private name of the token                                        | yes      |
| description | string | Private token description                                        | yes      |
| image       | string | Private uri to an image or additional metadata                   | yes      |

### <a name="nftdossier"></a>NftDossier
NftDossier returns all the information about a token that the viewer is permitted to view.  If no [viewer](#viewerinfo) is provided, NftDossier must only display the information that has been made public.  The response may include the owner, the public metadata, the private metadata, the reason the private metadata is not viewable, whether ownership is public, whether the private metadata is public, and (if the querier is the owner,) the approvals for this token as well as the inventory-wide approvals for the owner.

##### Request
```
{
	"nft_dossier": {
		"token_id": "ID_of_the_token_being_queried",
		"viewer": {
			"address": "address_of_the_querier_if_supplying_optional_ViewerInfo",
			"viewing_key": "viewer's_key_if_supplying_optional_ViewerInfo"
		},
		"include_expired": true | false
	}
}
```
| Name            | Type                                  | Description                                                           | Optional | Value If Omitted |
|-----------------|---------------------------------------|-----------------------------------------------------------------------|----------|------------------|
| token_id        | string                                | ID of the token being queried                                         | no       |                  |
| viewer          | [ViewerInfo (see above)](#viewerinfo) | The address and viewing key performing this query                     | yes      | nothing          |
| include_expired | bool                                  | True if expired approvals should be included in the response          | yes      | false            |

##### Response
```
{
	"nft_dossier": {
		"owner": "address_of_the_token_owner",
		"public_metadata": {
			"name": "optional_public_name_of_the_token",
			"description": "optional_public_description",
			"image": "optional_public_uri_containing_an_image_or_additional_metadata"
		},
		"private_metadata": {
			"name": "optional_private_name_of_the_token",
			"description": "optional_private_description",
			"image": "optional_private_uri_containing_an_image_or_additional_metadata"
		},
		"display_private_metadata_error": "optional_error_describing_why_private_metadata_is_not_viewable_if_applicable",
		"owner_is_public": true | false,
		"public_ownership_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
		"private_metadata_is_public": true | false,
		"private_metadata_is_public_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
		"token_approvals": [
			{
				"address": "whitelisted_address",
				"view_owner_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
				"view_private_metadata_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
				"transfer_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
			},
			{
				"...": "..."
			}
		],
		"inventory_approvals": [
			{
				"address": "whitelisted_address",
				"view_owner_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
				"view_private_metadata_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
				"transfer_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
			},
			{
				"...": "..."
			}
		]
	}
}
```
| Name                                  | Type                                                  | Description                                                                            | Optional | 
|---------------------------------------|-------------------------------------------------------|----------------------------------------------------------------------------------------|----------|
| owner                                 | string (HumanAddr)                                    | Address of the token's owner                                                           | yes      |
| public_metadata                       | [Metadata (see above)](#metadata)                     | The token's public metadata                                                            | yes      |
| private_metadata                      | [Metadata (see above)](#metadata)                     | The token's private metadata                                                           | yes      |
| display_private_metadata_error        | string                                                | If the private metadata is not displayed, the corresponding error message              | yes      |
| owner_is_public                       | bool                                                  | True if ownership is public for this token                                             | no       |
| public_ownership_expiration           | [Expiration (see above)](#expiration)                 | When public ownership expires for this token.  Can be a blockheight, time, or never    | yes      |
| private_metadata_is_public            | bool                                                  | True if private metadata is public for this token                                      | no       |
| private_metadata_is_public_expiration | [Expiration (see above)](#expiration)                 | When public display of private metadata expires.  Can be a blockheight, time, or never | yes      |
| token_approvals                       | array of [Snip721Approval (see below)](#snipapproval) | List of approvals for this token                                                       | yes      |
| inventory_approvals                   | array of [Snip721Approval (see below)](#snipapproval) | List of inventory-wide approvals for the token's owner                                 | yes      |

#### <a name="snipapproval"></a> Snip721Approval
The Snip721Approval object is used to display all the approvals (and their expirations) that have been granted to a whitelisted address.  The expiration field must be null if the whitelisted address does not have that corresponding permission type.
```
{
	"address": "whitelisted_address",
	"view_owner_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
	"view_private_metadata_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
	"transfer_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
}
```
| Name                             | Type                                  | Description                                                                                 | Optional | 
|----------------------------------|---------------------------------------|---------------------------------------------------------------------------------------------|----------|
| address                          | string (HumanAddr)                    | The whitelisted address                                                                     | no       |
| view_owner_expiration            | [Expiration (see above)](#expiration) | The expiration for view_owner permission.  Can be a blockheight, time, or never             | yes      |
| view_private_metadata_expiration | [Expiration (see above)](#expiration) | The expiration for view_private_metadata permission.  Can be a blockheight, time, or never | yes      |
| transfer_expiration              | [Expiration (see above)](#expiration) | The expiration for transfer permission.  Can be a blockheight, time, or never               | yes      |

### TokenApprovals
TokenApprovals returns whether the owner and private metadata of a token is public, and lists all the approvals specific to this token.  Only the token's owner may perform TokenApprovals.

##### Request
```
{
	"token_approvals": {
		"token_id": "ID_of_the_token_being_queried",
		"viewing_key": "the_token_owner's_viewing_key",
		"include_expired": true | false
	}
}
```
| Name            | Type   | Description                                                           | Optional | Value If Omitted |
|-----------------|--------|-----------------------------------------------------------------------|----------|------------------|
| token_id        | string | ID of the token being queried                                         | no       |                  |
| viewing_key     | string | The token owner's viewing key                                         | no       |                  |
| include_expired | bool   | True if expired approvals should be included in the response          | yes      | false            |

##### Response
```
{
	"token_approvals": {
		"owner_is_public": true | false,
		"public_ownership_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
		"private_metadata_is_public": true | false,
		"private_metadata_is_public_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
		"token_approvals": [
			{
				"address": "whitelisted_address",
				"view_owner_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
				"view_private_metadata_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
				"transfer_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
			},
			{
				"...": "..."
			}
		],
	}
}
```
| Name                                  | Type                                                  | Description                                                                            | Optional | 
|---------------------------------------|-------------------------------------------------------|----------------------------------------------------------------------------------------|----------|
| owner_is_public                       | bool                                                  | True if ownership is public for this token                                             | no       |
| public_ownership_expiration           | [Expiration (see above)](#expiration)                 | When public ownership expires for this token.  Can be a blockheight, time, or never    | yes      |
| private_metadata_is_public            | bool                                                  | True if private metadata is public for this token                                      | no       |
| private_metadata_is_public_expiration | [Expiration (see above)](#expiration)                 | When public display of private metadata expires.  Can be a blockheight, time, or never | yes      |
| token_approvals                       | array of [Snip721Approval (see above)](#snipapproval) | List of approvals for this token                                                       | no       |

### ApprovedForAll
ApprovedForAll displays all the addresses that have approval to transfer all of the specified owner's tokens.  This is provided to comply with CW-721 specification, but because approvals are private on  Secret Network, if the `owner`'s viewing key is not provided, no approvals should be displayed.  For a more complete list of inventory-wide approvals, the owner should use [InventoryApprovals](#inventoryapprovals) which also includes view_owner and view_private_metadata approvals.

##### Request
```
{
	"approved_for_all": {
		"owner": "address_whose_approvals_are_being_queried",
		"viewing_key": "owner's_viewing_key"
		"include_expired": true | false
	}
}
```
| Name            | Type               | Description                                                                              | Optional | Value If Omitted |
|-----------------|--------------------|------------------------------------------------------------------------------------------|----------|------------------|
| owner           | string (HumanAddr) | The address whose approvals are being queried                                            | no       |                  |
| viewing_key     | string             | The owner's viewing key                                                                  | yes      | nothing          |
| include_expired | bool               | True if expired approvals should be included in the response                             | yes      | false            |

##### Response
```
{
	"approved_for_all": {
		"operators": [
				{
					"spender": "address_with_transfer_approval",
					"expires": "never" | {"at_height": 999999} | {"at_time":999999}
				},
				{
					"...": "..."
				}
		]
	}
}
```
| Name      | Type                                                 | Description                                              | Optional | 
|-----------|------------------------------------------------------|----------------------------------------------------------|----------|
| operators | array of [Cw721Approval (see above)](#cw721approval) | List of approvals to transfer all of the owner's tokens  | no       |

### <a name="inventoryapprovals"></a> InventoryApprovals
InventoryApprovals returns whether all the address' tokens have public ownership and/or public display of private metadata, and lists all the inventory-wide approvals the address has granted.  Only the viewing key for this specified address should be accepted.

##### Request
```
{
	"inventory_approvals": {
		"address": "address_whose_approvals_are_being_queried",
		"viewing_key": "the_viewing_key_associated_with_this_address",
		"include_expired": true | false
	}
}
```
| Name            | Type               | Description                                                           | Optional | Value If Omitted |
|-----------------|--------------------|-----------------------------------------------------------------------|----------|------------------|
| address         | string (HumanAddr) | The address whose inventory-wide approvals are being queried          | no       |                  |
| viewing_key     | string             | The viewing key associated with this address                          | no       |                  |
| include_expired | bool               | True if expired approvals should be included in the response          | yes      | false            |

##### Response
```
{
	"inventory_approvals": {
		"owner_is_public": true | false,
		"public_ownership_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
		"private_metadata_is_public": true | false,
		"private_metadata_is_public_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
		"inventory_approvals": [
			{
				"address": "whitelisted_address",
				"view_owner_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
				"view_private_metadata_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
				"transfer_expiration": "never" | {"at_height": 999999} | {"at_time":999999},
			},
			{
				"...": "..."
			}
		],
	}
}
```
| Name                                  | Type                                                  | Description                                                                            | Optional | 
|---------------------------------------|-------------------------------------------------------|----------------------------------------------------------------------------------------|----------|
| owner_is_public                       | bool                                                  | True if ownership is public for all of this address' tokens                            | no       |
| public_ownership_expiration           | [Expiration (see above)](#expiration)                 | When public ownership expires for all tokens.  Can be a blockheight, time, or never    | yes      |
| private_metadata_is_public            | bool                                                  | True if private metadata is public for all of this address' tokens                     | no       |
| private_metadata_is_public_expiration | [Expiration (see above)](#expiration)                 | When public display of private metadata expires.  Can be a blockheight, time, or never | yes      |
| inventory_approvals                   | array of [Snip721Approval (see above)](#snipapproval) | List of inventory-wide approvals for this address                                      | no       |

### Tokens
Tokens displays an optionally paginated, lexicographically ordered list of all the token IDs that belong to the specified `owner`.  It must only display the owner's tokens on which the querier has view_owner permission.  If no viewing key is provided, it must only display the owner's tokens that have public ownership.  When paginating, supply the last token ID received in a response as the `start_after` string of the next query to continue listing where the previous query stopped.

##### Request
```
{
	"tokens": {
		"owner": "address_whose_inventory_is_being_queried",
		"viewer": "address_of_the_querier_if_different_from_owner",
		"viewing_key": "querier's_viewing_key"
		"start_after": "optional_string_where_results_will_only_be_token_IDs_that_come_later_in_lexicographical_order",
		"limit": 10
	}
}
```
| Name        | Type               | Description                                                                              | Optional | Value If Omitted       |
|-------------|--------------------|------------------------------------------------------------------------------------------|----------|------------------------|
| owner       | string (HumanAddr) | The address whose inventory is being queried                                             | no       |                        |
| viewer      | string (HumanAddr) | The querier's address if different from the `owner`                                      | yes      | nothing                |
| viewing_key | string             | The querier's viewing key                                                                | yes      | nothing                |
| start_after | string             | Results must only list token IDs that come after this string in lexicographical order    | yes      | nothing                |
| limit       | number (u32)       | Number of token IDs to return                                                            | yes      | developer's discretion |

##### Response
```
{
	"token_list": {
		"tokens": [
			"list", "of", "the", "owner's", "tokens", "..."
		]
	}
}
```
| Name    | Type            | Description                                                          | Optional | 
|---------|-----------------|----------------------------------------------------------------------|----------|
| tokens  | array of string | A list of token IDs owned by the specified `owner`                   | no       |

### TransactionHistory
TransactionHistory displays an optionally paginated list of transactions (mint, burn, and transfer) in reverse chronological order that involve the specified address.

##### Request
```
{
	"transaction_history": {
		"address": "address_whose_tx_history_is_being_queried",
		"viewing_key": "address'_viewing_key"
		"page": "optional_page_to_display",
		"page_size": 10
	}
}
```
| Name        | Type               | Description                                                                                                           | Optional | Value If Omitted       |
|-------------|--------------------|-----------------------------------------------------------------------------------------------------------------------|----------|------------------------|
| address     | string (HumanAddr) | The address whose transaction history is being queried                                                                | no       |                        |
| viewing_key | string             | The address' viewing key                                                                                              | no       |                        |
| page        | number (u32)       | The page number to display, where the first transaction shown skips the `page` * `page_size` most recent transactions | yes      | 0                      |
| page_size   | number (u32)       | Number of transactions to return                                                                                      | yes      | developer's discretion |

##### Response
```
{
	"transaction_history": {
		"txs": [
			{
				"tx_id": 9999,
				"blockheight": 999999,
				"token_id": "token_involved_in_the_tx",
				"action": {
					"transfer": {
						"from": "previous_owner_of_the_token",
						"sender": "address_that_sent_the_token_if_different_than_the_previous_owner",
						"recipient": "new_owner_of_the_token"
					}
				},
				"memo": "optional_memo_for_the_tx"
			},
			{
				"tx_id": 9998,
				"blockheight": 999998,
				"token_id": "token_involved_in_the_tx",
				"action": {
					"mint": {
						"minter": "address_that_minted_the_token",
						"recipient": "owner_of_the_newly_minted_token"
					}
				},
				"memo": "optional_memo_for_the_tx"
			},
			{
				"tx_id": 9997,
				"blockheight": 999997,
				"token_id": "token_involved_in_the_tx",
				"action": {
					"burn": {
						"owner": "previous_owner_of_the_token",
						"burner": "address_that_burned_the_token_if_different_than_the_previous_owner",
					}
				},
				"memo": "optional_memo_for_the_tx"
			},
			{
				"...": "..."
			}
		],
	}
}
```
| Name | Type                           | Description                                                                            | Optional | 
|------|--------------------------------|----------------------------------------------------------------------------------------|----------|
| txs  | array of [Tx (see below)](#tx) | list of transactions in reverse chronological order that involve the specified address | no       |

#### <a name="tx"></a> Tx
The Tx object contains all the information pertaining to a [mint](#txmint), [burn](#txburn), or [transfer](#txxfer) transaction.
```
{
	"tx_id": 9999,
	"blockheight": 999999,
	"token_id": "token_involved_in_the_tx",
	"action": { TxAction::Transfer | TxAction::Mint | TxAction::Burn },
	"memo": "optional_memo_for_the_tx"
}
```
| Name        | Type                              | Description                                                                               | Optional | 
|-------------|-----------------------------------|-------------------------------------------------------------------------------------------|----------|
| tx_id       | number (u64)                      | The transaction identifier                                                                | no       |
| blockheight | number (u64)                      | The number of the block that contains the transaction                                     | no       |
| token_id    | string                            | The token involved in the transaction                                                     | no       |
| action      | [TxAction (see below)](#txaction) | The type of transaction and the information specific to that type                         | no       |
| memo        | string                            | `memo` for the transaction that is only viewable by addresses involved in the transaction | yes      |

#### <a name="txaction"></a> TxAction
The TxAction object defines the type of transaction and holds the information specific to that type.

* <a name="txmint"></a>TxAction::Mint
```
{
	"minter": "address_that_minted_the_token",
	"recipient": "owner_of_the_newly_minted_token"
}

```
| Name      | Type               | Description                                                                    | Optional | 
|-----------|--------------------|--------------------------------------------------------------------------------|----------|
| minter    | string (HumanAddr) | The address that minted the token                                              | no       |
| recipient | string (HumanAddr) | The address of the newly minted token's owner                                  | no       |

* <a name="txxfer"></a>TxAction::Transfer
```
{
	"from": "previous_owner_of_the_token",
	"sender": "address_that_sent_the_token_if_different_than_the_previous_owner",
	"recipient": "new_owner_of_the_token"
}

```
| Name      | Type               | Description                                                                    | Optional | 
|-----------|--------------------|--------------------------------------------------------------------------------|----------|
| from      | string (HumanAddr) | The previous owner of the token                                                | no       |
| sender    | string (HumanAddr) | The address that sent the token if different than the previous owner           | yes      |
| recipient | string (HumanAddr) | The new owner of the token                                                     | no       |

* <a name="txburn"></a>TxAction::Burn
```
{
	"owner": "previous_owner_of_the_token",
	"burner": "address_that_burned_the_token_if_different_than_the_previous_owner",
}

```
| Name      | Type               | Description                                                                    | Optional | 
|-----------|--------------------|--------------------------------------------------------------------------------|----------|
| owner     | string (HumanAddr) | The previous owner of the token                                                | no       |
| burner    | string (HumanAddr) | The address that burned the token if different than the previous owner         | yes      |

# <a name="receiver"></a>Receiver Interface
When the SNIP-721 contract executes [SendNft](#sendnft) and [BatchSendNft](#batchsend) messages, it must perform a callback to the receiving contract's receiver interface if the contract had registered its code hash using [RegisterReceiveNft](#registerreceive).  [BatchReceiveNft](#batchreceivenft) is preferred over [ReceiveNft](#receivenft), because ReceiveNft does not allow the recipient to know who sent the token, only its previous owner, and ReceiveNft can only process one token.  So it is inefficient when sending multiple tokens to the same contract (a deck of game cards for instance).  ReceiveNft primarily exists just to maintain CW-721 compliance.

<a name="cwsender"></a>
Also, it should be noted that the CW-721 `sender` field is inaccurately named, because it is used to hold the address the token came from, not the address that sent it (which is not always the same).  The name is reluctantly kept in [ReceiveNft](#receivenft) to maintain CW-721 compliance, but [BatchReceiveNft](#batchreceivenft) uses `sender` to hold the sending address (which matches both its true role and its SNIP-20 Receive counterpart).  Any contract that is implementing both Receiver Interfaces must be sure that the ReceiveNft `sender` field is actually processed like a BatchReceiveNft `from` field.  Again, apologies for any confusion caused by propagating inaccuracies, but because [InterNFT](https://internft.org) is planning on using CW-721 standards, compliance with CW-721 might be necessary.

## <a name="receivenft"></a>ReceiveNft
ReceiveNft may be a HandleMsg variant of any contract that wants to implement a receiver interface.  [BatchReceiveNft](#batchreceivenft), which is more informative and more efficient, is preferred over ReceiveNft.  
```
{
	"receive_nft": {
		"sender": "address_of_the_previous_owner_of_the_token",
		"token_id": "ID_of_the_sent_token",
		"msg": "optional_base64_encoded_Binary_message_used_to_control_receiving_logic"
	}
}
```
| Name     | Type                           | Description                                                                                            | Optional | Value If Omitted |
|----------|--------------------------------|--------------------------------------------------------------------------------------------------------|----------|------------------|
| sender   | string (HumanAddr)             | Address of the token's previous owner ([see above](#cwsender) about this inaccurate naming convention) | no       |                  |
| token_id | string                         | ID of the sent token                                                                                   | no       |                  |
| msg      | string (base64 encoded Binary) | Msg used to control receiving logic                                                                    | yes      | nothing          |

## <a name="batchreceivenft"></a>BatchReceiveNft
BatchReceiveNft may be a HandleMsg variant of any contract that wants to implement a receiver interface.  BatchReceiveNft, which is more informative and more efficient, is preferred over [ReceiveNft](#receivenft).
```
{
	"batch_receive_nft": {
		"sender": "address_that_sent_the_tokens",
		"from": "address_of_the_previous_owner_of_the_tokens",
		"token_ids": [
			"list", "of", "tokens", "sent", "..."
		],
		"msg": "optional_base64_encoded_Binary_message_used_to_control_receiving_logic"
	}
}
```
| Name      | Type                           | Description                                                                                                              | Optional | Value If Omitted |
|-----------|--------------------------------|--------------------------------------------------------------------------------------------------------------------------|----------|------------------|
| sender    | string (HumanAddr)             | Address that sent the tokens (this field has no ReceiveNft equivalent, [see above](#cwsender))                           | no       |                  |
| from      | string (HumanAddr)             | Address of the tokens' previous owner (this field is equivalent to the ReceiveNft `sender` field, [see above](#cwsender))| no       |                  |
| token_ids | array of string                | List of the tokens sent                                                                                                  | no       |                  |
| msg       | string (base64 encoded Binary) | Msg used to control receiving logic                                                                                      | yes      | nothing          |

# Optional Functionality
These messages and queries are not mandatory for SNIP-721 compliance; however, they are considered useful enough to be included in the [SNIP-721 reference implementation](https://github.com/baedrik/snip721-reference-impl).  As such, the SNIP-721 toolkit package (TODO add link), includes functions to call these messages and queries, and SNIP-721 contract developers may wish to implement them.

## Minting and Modifying Tokens

## Messages

### MintNft
MintNft mints a single token.

##### Request
```
{
	"mint_nft": {
		"token_id": "optional_ID_of_new_token",
		"owner": "optional_address_the_new_token_will_be_minted_to",
		"public_metadata": {
			"name": "optional_public_name",
			"description": "optional_public_text_description",
			"image": "optional_public_uri_pointing_to_an_image_or_additional_off-chain_metadata"
		},
		"private_metadata": {
			"name": "optional_private_name",
			"description": "optional_private_text_description",
			"image": "optional_private_uri_pointing_to_an_image_or_additional_off-chain_metadata"
		},
		"memo": "optional_memo_for_the_mint_tx",
		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name             | Type                                   | Description                                                                                   | Optional | Value If Omitted     |
|------------------|----------------------------------------|-----------------------------------------------------------------------------------------------|----------|----------------------|
| token_id         | string                                 | Identifier for the token to be minted                                                         | yes      | minting order number |
| owner            | string (HumanAddr)                     | Address of the owner of the minted token                                                      | yes      | env.message.sender   |
| public_metadata  | [Metadata (see above)](#metadata)      | The metadata that is publicly viewable                                                        | yes      | nothing              |
| private_metadata | [Metadata (see above)](#metadata)      | The metadata that is viewable only by the token owner and addresses the owner has whitelisted | yes      | nothing              |
| memo             | string                                 | `memo` for the mint tx that is only viewable by addresses involved in the mint (minter, owner)| yes      | nothing              |
| padding          | string                                 | An ignored string that can be used to maintain constant message length                        | yes      | nothing              |

##### Response
```
{
	"mint_nft": {
		"token_id": "ID_of_minted_token",
	}
}
```
The ID of the minted token should also be returned in a LogAttribute with the key `minted`.

### AddMinters
AddMinters must add the provided addresses to the list of authorized minters.  This should only be callable by an admin address.

##### Request
```
{
	"add_minters": {
		"minters": [
			"list", "of", "addresses", "to", "add", "to", "the", "list", "of", "minters", "..."
		],
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name    | Type                        | Description                                                                                                        | Optional | Value If Omitted |
|---------|-----------------------------|--------------------------------------------------------------------------------------------------------------------|----------|------------------|
| minters | array of string (HumanAddr) | The list of addresses to add to the list of authorized minters                                                     | no       |                  |
| padding | string                      | An ignored string that can be used to maintain constant message length                                             | yes      | nothing          |

##### Response
```
{
	"add_minters": {
		"status": "success"
	}
}
```

### RemoveMinters
RemoveMinters must remove the provided addresses from the list of authorized minters.  This should only be callable by an admin address.

##### Request
```
{
	"remove_minters": {
		"minters": [
			"list", "of", "addresses", "to", "remove", "from", "the", "list", "of", "minters", "..."
		],
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name    | Type                        | Description                                                                                                        | Optional | Value If Omitted |
|---------|-----------------------------|--------------------------------------------------------------------------------------------------------------------|----------|------------------|
| minters | array of string (HumanAddr) | The list of addresses to remove from the list of authorized minters                                                | no       |                  |
| padding | string                      | An ignored string that can be used to maintain constant message length                                             | yes      | nothing          |

##### Response
```
{
	"remove_minters": {
		"status": "success"
	}
}
```

### SetMinters
SetMinters must precisely define the list of authorized minters.  This should only be callable by an admin address.

##### Request
```
{
	"set_minters": {
		"minters": [
			"list", "of", "addresses", "that", "have", "minting", "authority", "..."
		],
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name    | Type                        | Description                                                                                 | Optional | Value If Omitted |
|---------|-----------------------------|---------------------------------------------------------------------------------------------|----------|------------------|
| minters | array of string (HumanAddr) | The list of addresses to are allowed to mint                                                | no       |                  |
| padding | string                      | An ignored string that can be used to maintain constant message length                      | yes      | nothing          |

##### Response
```
{
	"set_minters": {
		"status": "success"
	}
}
```

### SetPublicMetadata
SetPublicMetadata will set the public metadata to the input metadata.  The SNIP-721 contract may want to limit which addresses have the authority to alter the public metadata.  See [here](https://github.com/baedrik/snip721-reference-impl/blob/master/README.md#setpublic) for a description of which addresses are given authority to alter public metadata in the [reference implementation](https://github.com/baedrik/snip721-reference-impl)

##### Request
```
{
	"set_public_metadata": {
		"token_id": "ID_of_token_whose_metadata_should_be_updated",
		"metadata": {
			"name": "optional_public_name",
			"description": "optional_public_text_description",
			"image": "optional_public_uri_pointing_to_an_image_or_additional_off-chain_metadata"
		},
		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name     | Type                                 | Description                                                            | Optional | Value If Omitted |
|----------|--------------------------------------|------------------------------------------------------------------------|----------|------------------|
| token_id | string                               | ID of the token whose metadata should be updated                       | no       |                  |
| metadata | [Metadata (see above)](#metadata)    | The new public metadata for the token                                  | no       |                  |
| padding  | string                               | An ignored string that can be used to maintain constant message length | yes      | nothing          |

##### Response
```
{
	"set_public_metadata": {
		"status": "success"
	}
}
```

### SetPrivateMetadata
SetPrivateMetadata will set the private metadata to the input metadata.  The SNIP-721 contract may want to limit which addresses have the authority to alter the private metadata.  See [here](https://github.com/baedrik/snip721-reference-impl/blob/master/README.md#setprivate) for a description of which addresses are given authority to alter private metadata in the [reference implementation](https://github.com/baedrik/snip721-reference-impl)

##### Request
```
{
	"set_private_metadata": {
		"token_id": "ID_of_token_whose_metadata_should_be_updated",
		"metadata": {
			"name": "optional_private_name",
			"description": "optional_private_text_description",
			"image": "optional_private_uri_pointing_to_an_image_or_additional_off-chain_metadata"
		},
		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name     | Type                                 | Description                                                            | Optional | Value If Omitted |
|----------|--------------------------------------|------------------------------------------------------------------------|----------|------------------|
| token_id | string                               | ID of the token whose metadata should be updated                       | no       |                  |
| metadata | [Metadata (see above)](#metadata)    | The new private metadata for the token                                 | no       |                  |
| padding  | string                               | An ignored string that can be used to maintain constant message length | yes      | nothing          |

##### Response
```
{
	"set_private_metadata": {
		"status": "success"
	}
}
```

## Queries

### Minters
Minters returns the list of addresses that are authorized to mint tokens.  This query is not authenticated.

##### Request
```
{
	"minters": {}
}
```
##### Response
```
{
	"minters": {
		“minters”: [
			"list", "of", "authorized", "minters", "..."
		]
	}
}
```
| Name    | Type                        | Description                              | Optional | 
|---------|-----------------------------|------------------------------------------|----------|
| minters | array of string (HumanAddr) | List of addresses with minting authority | no       |

## Batch Processing
These messages are used to process multiple token transactions in a single Cosmos message.  By avoiding the overhead of creating an additional Cosmos message for every token, an application can save 70k-80k gas per additional token processed.  This may be particularly useful in gaming applications where entire decks of "cards" are being processed.

## Messages

### BatchMintNft
BatchMintNft mints a list of tokens.

##### Request
```
{
	"batch_mint_nft": {
		"mints": [
			{
				"token_id": "optional_ID_of_new_token",
				"owner": "optional_address_the_new_token_will_be_minted_to",
				"public_metadata": {
					"name": "optional_public_name",
					"description": "optional_public_text_description",
					"image": "optional_public_uri_pointing_to_an_image_or_additional_off-chain_metadata"
				},
				"private_metadata": {
					"name": "optional_private_name",
					"description": "optional_private_text_description",
					"image": "optional_private_uri_pointing_to_an_image_or_additional_off-chain_metadata"
				},
				"memo": "optional_memo_for_the_mint_tx"
			},
			{
				"...": "..."
			}
		],
		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name    | Type                                  | Description                                                            | Optional | Value If Omitted |
|---------|---------------------------------------|------------------------------------------------------------------------|----------|------------------|
| mints   | array of [Mint (see below)](#mint)    | A list of all the mint operations to perform                           | no       |                  |
| padding | string                                | An ignored string that can be used to maintain constant message length | yes      | nothing          |

##### Response
```
{
	"batch_mint_nft": {
		"token_ids": [
			"IDs", "of", "tokens", "that", "were", "minted", "..."
		]
	}
}
```
The IDs of the minted tokens should also be returned in a LogAttribute with the key `minted`.

#### <a name="mint"></a>Mint
The Mint object defines the data necessary to mint one token.
```
{
	"token_id": "optional_ID_of_new_token",
	"owner": "optional_address_the_new_token_will_be_minted_to",
	"public_metadata": {
		"name": "optional_public_name",
		"description": "optional_public_text_description",
		"image": "optional_public_uri_pointing_to_an_image_or_additional_off-chain_metadata"
	},
	"private_metadata": {
		"name": "optional_private_name",
		"description": "optional_private_text_description",
		"image": "optional_private_uri_pointing_to_an_image_or_additional_off-chain_metadata"
	},
	"memo": "optional_memo_for_the_mint_tx"
}
```
| Name             | Type                                 | Description                                                                                        | Optional | Value If Omitted     |
|------------------|--------------------------------------|----------------------------------------------------------------------------------------------------|----------|----------------------|
| token_id         | string                               | Identifier for the token to be minted                                                              | yes      | minting order number |
| owner            | string (HumanAddr)                   | Address of the owner of the minted token                                                           | yes      | env.message.sender   |
| public_metadata  | [Metadata (see above)](#metadata)    | The metadata that is publicly viewable                                                             | yes      | nothing              |
| private_metadata | [Metadata (see above)](#metadata)    | The metadata that is viewable only by the token owner and addresses the owner has whitelisted      | yes      | nothing              |
| memo             | string                               | `memo` for the mint tx that is only viewable by addresses involved in the mint (minter, owner)     | yes      | nothing              |

### BatchTransferNft
BatchTransferNft is used to perform multiple token transfers.  The message sender may specify a list of tokens to transfer to one `recipient` address in each [Transfer](#transfer) object, and any `memo` provided must be applied to every token transferred in that one `Transfer` object.  The message sender may provide multiple `Transfer` objects to perform transfers to multiple addresses, providing a different `memo` for each address if desired.  Each individual transfer of a token must show separately in transaction histories.  The message sender must have permission to transfer all the tokens listed (either by being the owner or being granted transfer approval) and every listed `token_id` must be valid.  Any token that is transferred to a new owner must have its single-token approvals cleared.

##### Request
```
{
	"batch_transfer_nft": {
		"transfers": [
			{
				"recipient": "address_receiving_the_tokens",
				"token_ids": [
					"list", "of", "token", "IDs", "to", "transfer"
				],
				"memo": "optional_memo_applied_to_the_transfer_tx_for_every_token_listed_in_this_Transfer_object"
			},
			{
				"...": "..."
			}
		],
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name      | Type                                          | Description                                                                                          | Optional | Value If Omitted |
|-----------|-----------------------------------------------|------------------------------------------------------------------------------------------------------|----------|------------------|
| transfers | array of [Transfer (see below)](#transfer)    | List of `Transfer` objects to process                                                                | no       |                  |
| padding   | string                                        | An ignored string that can be used to maintain constant message length                               | yes      | nothing          |

##### Response
```
{
	"batch_transfer_nft": {
		"status": "success"
	}
}
```

#### <a name="transfer"></a>Transfer
The Transfer object provides a list of tokens to transfer to one `recipient` address, as well as an optional `memo` that would be included with every logged token transfer.
```
{
	"recipient": "address_receiving_the_tokens",
	"token_ids": [
		"list", "of", "token", "IDs", "to", "transfer", "..."
	],
	"memo": "optional_memo_applied_to_the_transfer_tx_for_every_token_listed_in_this_Transfer_object"
}
```
| Name      | Type               | Description                                                                                                                         | Optional | Value If Omitted |
|-----------|--------------------|-------------------------------------------------------------------------------------------------------------------------------------|----------|------------------|
| recipient | string (HumanAddr) | Address receiving the listed tokens                                                                                                 | no       |                  |
| token_ids | array of string    | List of token IDs to transfer to the `recipient`                                                                                    | no       |                  |
| memo      | string             | `memo` for the transfer transactions that is only viewable by addresses involved in the transfer (recipient, sender, previous owner)| yes      | nothing          |

### <a name="batchsend"></a>BatchSendNft
BatchSendNft is used to perform multiple token transfers, and then call the recipient contracts' [BatchReceiveNft](#batchreceivenft) (or [ReceiveNft](#receivenft)) if they have registered their receiver interface with the NFT contract.  The message sender may specify a list of tokens to send to one recipient address in each [Send](#send) object, and any `memo` or `msg` provided must be applied to every token transferred in that one `Send` object.  If the list of transferred tokens belonged to multiple previous owners, a separate BatchReceiveNft callback must be performed for each of the previous owners.  If the contract only implements ReceiveNft, one ReceiveNft must be performed for every sent token.  Therefore it is highly recommended to implement BatchReceiveNft if there is the possibility of being sent multiple tokens at one time.  This will significantly reduce gas costs.  

The message sender may provide multiple [Send](#send) objects to perform sends to multiple addresses, providing a different `memo` and `msg` for each address if desired.  Each individual transfer of a token must show separately in transaction histories.  The message sender must have permission to transfer all the tokens listed (either by being the owner or being granted transfer approval) and every token ID must be valid.  Any token that is transferred to a new owner must have its single-token approvals cleared.
If any BatchReceiveNft (or ReceiveNft) callback fails, the entire transaction must be reverted (even the transfers must not take place).

##### Request
```
{
	"batch_send_nft": {
		"sends": [
			{
				"contract": "address_receiving_the_tokens",
				"token_ids": [
					"list", "of", "token", "IDs", "to", "transfer", "..."
				],
				"msg": "optional_base64_encoded_Binary_message_sent_with_every_BatchReceiveNft_callback_made_for_this_one_Send_object",
				"memo": "optional_memo_applied_to_the_transfer_tx_for_every_token_listed_in_this_Send_object"
			},
			{
				"...": "..."
			}
		],
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name      | Type                                  | Description                                                                                                        | Optional | Value If Omitted |
|-----------|---------------------------------------|--------------------------------------------------------------------------------------------------------------------|----------|------------------|
| sends     | array of [Send (see below)](#send)    | List of `Send` objects to process                                                                                  | no       |                  |
| padding   | string                                | An ignored string that can be used to maintain constant message length                                             | yes      | nothing          |

##### Response
```
{
	"batch_send_nft": {
		"status": "success"
	}
}
```

#### <a name="send"></a>Send
The Send object provides a list of tokens to transfer to one recipient address, as well as an optional `memo` that would be included with every logged token transfer, and an optional `msg` that would be included with every [BatchReceiveNft](#batchreceivenft) or [ReceiveNft](#receivenft) callback made as a result of this Send object.  While Send keeps the `contract` field name in order be consistent with CW-721 specification, Secret Network does not have the same limitations as Cosmos, and it is possible to use [BatchSendNft](#batchsend) to transfer token ownership to a personal address (not a contract) or to a contract that does not implement any [Receiver Interface](#receiver).
```
{
	"contract": "address_receiving_the_tokens",
	"token_ids": [
		"list", "of", "token", "IDs", "to", "transfer", "..."
	],
	"msg": "optional_base64_encoded_Binary_message_sent_with_every_BatchReceiveNft_callback_made_for_this_one_Send_object",
	"memo": "optional_memo_applied_to_the_transfer_tx_for_every_token_listed_in_this_Send_object"
}
```
| Name      | Type                           | Description                                                                                                                 | Optional | Value If Omitted |
|-----------|--------------------------------|-----------------------------------------------------------------------------------------------------------------------------|----------|------------------|
| contract  | string (HumanAddr)             | Address receiving the listed tokens                                                                                         | no       |                  |
| token_ids | array of string                | List of token IDs to send to the recipient                                                                                  | no       |                  |
| msg       | string (base64 encoded Binary) | `msg` included with every BatchReceiveNft (or ReceiveNft) callback performed for this `Send` object                         | yes      | nothing          |
| memo      | string                         | `memo` for the transfer txs that is only viewable by addresses involved in the transfer (recipient, sender, previous owner) | yes      | nothing          |

## Burning
These message standards are provided for contracts that may wish to allow burning tokens.

## Messages

### BurnNft
BurnNft is used to burn a single token, providing an optional `memo` to include in the burn's transaction history if desired.  The message sender must have permission to burn the token.  Implementation details regarding who is authorized to burn tokens is left to the contract developer.  The [reference implementation](https://github.com/baedrik/snip721-reference-impl) allows only the token owner and anyone else with valid transfer approval to burn this token.

##### Request
```
{
	"burn_nft": {
		"token_id": "ID_of_the_token_to_burn",
		"memo": "optional_memo_for_the_burn_tx",
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name      | Type               | Description                                                                                                                         | Optional | Value If Omitted |
|-----------|--------------------|-------------------------------------------------------------------------------------------------------------------------------------|----------|------------------|
| token_id  | string             | Identifier of the token to burn                                                                                                     | no       |                  |
| memo      | string             | `memo` for the burn tx that is only viewable by addresses involved in the burn (message sender and previous owner if different)     | yes      | nothing          |
| padding   | string             | An ignored string that can be used to maintain constant message length                                                              | yes      | nothing          |

##### Response
```
{
	"burn_nft": {
		"status": "success"
	}
}
```

### BatchBurnNft
BatchBurnNft is used to burn multiple tokens.  The message sender may specify a list of tokens to burn in each [Burn](#burn) object, and any `memo` provided must be applied to every token burned in that one `Burn` object.  The message sender will usually list every token to be burned in one `Burn` object, but if a different `memo` is needed for different tokens being burned, multiple `Burn` objects may be listed. Each individual burning of a token must show separately in transaction histories.  The message sender must have permission to burn all the tokens listed.  Implementation details regarding who is authorized to burn tokens is left to the contract developer.  The [reference implementation](https://github.com/baedrik/snip721-reference-impl) allows only the token owner and anyone else with valid transfer approval to burn a token.

##### Request
```
{
	"batch_burn_nft": {
		"burns": [
			{
				"token_ids": [
					"list", "of", "token", "IDs", "to", "burn"
				],
				"memo": "optional_memo_applied_to_the_burn_tx_for_every_token_listed_in_this_Burn_object"
			},
			{
				"...": "..."
			}
		],
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name      | Type                                  | Description                                                                                                        | Optional | Value If Omitted |
|-----------|---------------------------------------|--------------------------------------------------------------------------------------------------------------------|----------|------------------|
| burns     | array of [Burn (see below)](#burn)    | List of `Burn` objects to process                                                                                  | no       |                  |
| padding   | string                                | An ignored string that can be used to maintain constant message length                                             | yes      | nothing          |

##### Response
```
{
	"batch_burn_nft": {
		"status": "success"
	}
}
```

#### <a name="burn"></a>Burn
The Burn object provides a list of tokens to burn, as well as an optional `memo` that would be included with every token burn transaction history.
```
{
	"token_ids": [
		"list", "of", "token", "IDs", "to", "burn", "..."
	],
	"memo": "optional_memo_applied_to_the_burn_tx_for_every_token_listed_in_this_Burn_object"
}
```
| Name      | Type            | Description                                                                                                                      | Optional | Value If Omitted |
|-----------|-----------------|----------------------------------------------------------------------------------------------------------------------------------|----------|------------------|
| token_ids | array of string | List of token IDs to burn                                                                                                        | no       |                  |
| memo      | string          | `memo` for the burn txs that is only viewable by addresses involved in the burn (message sender and previous owner if different) | yes      | nothing          |

## Making the Owner and/or Private Metadata Public
A SNIP-721 contract may wish to allow an owner to make a token's owner and/or its private metadata public.  It may also choose to allow an address to make all their tokens' owner and/or private metadata public.  The [reference implementation](https://github.com/baedrik/snip721-reference-impl) does this using the same convention as [SetWhitelistedApproval](#setwhitelisted).

## Messages

### <a name="setglobal"></a>SetGlobalApproval
The owner of a token can use SetGlobalApproval to make ownership and/or private metadata viewable by everyone.  This can be set for a single token or for an owner's entire inventory of tokens by choosing the appropriate [AccessLevel](#accesslevel).  SetGlobalApproval can also be used to revoke any global approval previously granted.

##### Request
```
{
	"set_global_approval": {
		"token_id": "optional_ID_of_the_token_to_grant_or_revoke_approval_on",
		"view_owner": "approve_token" | "all" | "revoke_token" | "none",
		"view_private_metadata": "approve_token" | "all" | "revoke_token" | "none",
		"expires": "never" | {"at_height": 999999} | {"at_time":999999},
 		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name                  | Type                                       | Description                                                                                        | Optional | Value If Omitted |
|-----------------------|--------------------------------------------|----------------------------------------------------------------------------------------------------|----------|------------------|
| token_id              | string                                     | If supplying either `approve_token` or `revoke_token` access, the token whose privacy is being set | yes      | nothing          |
| view_owner            | [AccessLevel (see above)](#accesslevel)    | Grant or revoke everyone's permission to view the ownership of a token/inventory                   | yes      | nothing          |
| view_private_metadata | [AccessLevel (see above)](#accesslevel)    | Grant or revoke everyone's permission to view the private metadata of a token/inventory            | yes      | nothing          |
| expires               | [Expiration (see above)](#expiration)      | Expiration of any approval granted in this message.  Can be a blockheight, time, or never          | yes      | "never"          |
| padding               | string                                     | An ignored string that can be used to maintain constant message length                             | yes      | nothing          |

##### Response
```
{
	"set_global_approval": {
		"status": "success"
	}
}
```

## Lootboxes and Wrapped Cards
The [reference implementation](https://github.com/baedrik/snip721-reference-impl) provides a [configuration setting](https://github.com/baedrik/snip721-reference-impl/blob/master/README.md#config) that prevents anyone, even the owner, from viewing or altering the private metadata of a token until it is unwrapped by calling [Reveal](#reveal).  This enables creating lootboxes or sealed "cards" that have already determined contents instead of randomly determining the contents when they are unwrapped.  If a developer chooses to implement this Seal/Reveal functionality in their own SNIP-721 contract, they may also wish to follow the same message formats.

## Messages

### <a name="reveal"></a>Reveal
Reveal unwraps the sealed private metadata, irreversibly marking the token as unwrapped.

##### Request
```
{
	"reveal": {
		"token_id": "ID_of_the_token_to_unwrap",
		"padding": "optional_ignored_string_that_can_be_used_to_maintain_constant_message_length"
	}
}
```
| Name     | Type                 | Description                                                            | Optional | Value If Omitted |
|----------|----------------------|------------------------------------------------------------------------|----------|------------------|
| token_id | string               | ID of the token to unwrap                                              | no       |                  |
| padding  | string               | An ignored string that can be used to maintain constant message length | yes      | nothing          |

##### Response
```
{
	"reveal": {
		"status": "success"
	}
}
```

## Queries

### IsUnwrapped
IsUnwrapped indicates whether the token has been unwrapped.  This query is not authenticated.

##### Request
```
{
	"is_unwrapped": {
		"token_id": "ID_of_the_token_whose_unwrapped_status_is_being_queried"
	}
}
```
| Name        | Type   | Description                                                                              | Optional | Value If Omitted |
|-------------|--------|------------------------------------------------------------------------------------------|----------|------------------|
| token_id    | string | The ID of the token whose unwrapped status is being queried                              | no       |                  |

##### Response
```
{
	"is_unwrapped": {
		"token_is_unwrapped": true | false
	}
}
```
| Name                | Type | Description                                                          | Optional | 
|---------------------|------|----------------------------------------------------------------------|----------|
| token_is_unwrapped  | bool | True if the token is unwrapped                                       | no       |
