# SNIP-20 Spec: Private, Fungible Tokens

SNIP-20, (or Secret-20) is a specification for private fungible tokens based on CosmWasm on the Secret Network. The name and design is loosely based on Ethereum's ERC20 standard, and is a superset of CosmWasm's CW20. The additions to this spec over the CW20 specification are mostly for privacy focused features, and as such will strive to maintain compatability. 

The specification is split into multiple sections, a contract may only implement some of this functionality, but must implement the base.

This handles balances and transfers. Note that all amounts are handled as Uint128 (128 bit integers with JSON string representation). Handling decimals is left to the UI and not interpreted

Contracts that implement this interface SHOULD support only a single token/symbol, and MUST conform to the standards set in this memo

## Notation

Notation in this document conforms to the standards set by [RFC-2119](http://microformats.org/wiki/rfc-2119)

## Terms

- __Message__ - This is an on-chain interface. It is triggered by sending a transaction, and receiving an on-chain response which is read by the client. Messages are authenticated both by the blockchain, and by the secret enclave
- __Query__ - This is an off-chain interface. Queries are done by returning data that a node has locally, and are not public. Query responses are returned immediately, and do not have to wait for blocks. In addition, queries cannot be authenticated using the standard interfaces. Any contract that wishes to strongly enforce query permissions must implement it themselves. (TBD - should this be standardized?)

## Padding

Secret-20 tokens may want to enforce constant length messages to avoid leaking data. To support this functionality, an optional _padding_ field may be sent with ANY of the messages in this spec. Contracts and Clients MUST ignore this field if sent, either in the request or response fields

# Sections

# Base

## Messages

### Transfer 
Moves tokens from the account that appears in the Cosmos message sender[1] field to the account in the recipient field. 

* Variation from CW-20: This command will work when trying to send funds to contract accounts as well.

##### Request parameters
	
	recipient - string. Contracts MUST validate that this parameter is in the form of a valid bech32 account address
	amount - Uint128. The amount of tokens to transfer

#### Response
	Will return a positive response if successful, or an error if failed.

##### Response parameters
	None

### Send
Moves amount from the Cosmos message sender[1] account to the recipient account, with the ability to register a receiver interface which will be called _after_ the transfer is completed. Contract MAY be an address of a contract that registered the Receiver interface. Msg is binary encoded data that can be decoded into a contract-specific message by the receiver, which will be passed to the recipient, along with the amount.
Even if the receiver function fails for any reason, the actual transfer of tokens MUST NOT be reverted.

##### Request parameters
    recipient - string. Contracts MUST validate that this parameter is in the form of a valid bech32 account address
	amount - Uint128. The amount of tokens to send
	msg? - 

### Burn
MUST remove amount tokens from the balance of Cosmos message sender[1] and MUST reduce the total supply by the same amount. 

##### Request parameters
	amount - Uint128. The amount of tokens to burn

### RegisterReceive
This message is used to tell the Secret-20 contract to call the _Receive_ function of the Cosmos message sender[1] after a successful _send_. 

In Secret Network this is used to pair a code hash with the contract address that must be called. This means that the Secret-20 MUST store the sent code_hash and use it when calling the _Receive_ function

TBD - is there any value in allowing an address other than msg.sender? i.e. If I wanted every time someone sends me tokens to trigger some _other_ contract.

##### Request parameters
	code_hash - string

### CreateViewingKey
This function _generates a new_ viewing key for Cosmos message sender[1], which is used in ALL account specific queries. This key is used to validate the identity of the caller, as in queries in cosmos there is no way to cryptographically authenticate the caller.
	
The Viewing Key MUST be stored in such a way that lookup takes a significant amount to time to perform, in order to be resistent to brute-force attacks.
The viewing key MUST NOT control any functions which actively affect the balance of the user.
	
##### Request Parameters
	
	entropy - string. A user supplied string used for entropy for generation of the viewing key. Secure implementation is left to the client, but it is recommended to use base-64 encoded random bytes and not predictable inputs
	
##### Response

	log:
	
	Key: "api_key"
	Value: Viewing key in the form: `api_key_[a-zA-Z0-9]{24}`

### SetViewingKey

Set a viewing key with a predefined value for Cosmos message sender[1], _without_ creating it. This is useful to manage multiple Secret-20 tokens using the same viewing key.

If a viewing key is already set, the contract MUST replace the current key. 
If a viewing key is not set, the contract MUST set the provided key as the viewing key.

It is NOT RECOMMENDED to use this function to create easy to remember passwords for users, but this is left up to implementors to enforce

##### Request Parameters
	
	viewing_key - string. A user supplied string that will either replace


## Queries

Queries are off-chain requests, that are not cryptographically validated. This means that contracts that wish to validate the caller of a query MUST implement some sort of authentication. Secret-20 uses an "API key" scheme, which validates a <viewing key, account> pair.

Authentication MUST happen on each query that reveals private account-specific information.
Authentication MUST be a resource intensive operation, that takes a significant[2] amount of time to compute. This is because such queries are open to offline brute-force attacks, which can be parallelized to scale linearly with the resources of a motivated attacker.
Authentication MUST perform the same computation even if the user does not have a viewing key set. 
Authentication response MUST be indistinguishable for both the case of a wrong viewing key and the case of a non-existant viewing key

### Balance - Authenticated
Returns the balance of the given address. Returns "0" if the address is unknown to the contract.

// tbd - specify exact response for viewing key failure?

##### Request parameters

	address - string. Contracts MUST validate that this parameter is in the form of a valid bech32 account address
	viewing_key - string


### TokenInfo - Unauthenticated
Returns the token info of the contract. The response MUST contain: token name, token symbol, and the number of decimals the token uses. The response MAY additionally contain the total_supply of tokens. This is to enable Layer-2 tokens which want to hide the amounts converted as well

##### Request parameters

None

##### Response

### TransferHistory - Authenticated

	address - string. Contracts MUST validate that this parameter is in the form of a valid bech32 account address
	viewing_key - string
	n - u32. Number of transactions to show, starting from the latest. I.e. n=1 will return only the latest transaction

# Allowances
A contract may allow actors to delegate some of their balance to other accounts. This is not as essential as with ERC20 as we use Send/Receive to send tokens to a contract, not Approve/TransferFrom. But it is still a nice use-case, and you can see how the Cosmos SDK wants to add payment allowances to native tokens. This is mainly designed to provide access to other public-key-based accounts.

There was an issue with race conditions in the original ERC20 approval spec. If you had an approval of 50 and I then want to reduce it to 20, I submit a Tx to set the allowance to 20. If you see that and immediately submit a tx using the entire 50, you then get access to the other 20. Not only did you quickly spend the 50 before I could reduce it, you get another 20 for free.

The solution discussed in the Ethereum community was an IncreaseAllowance and DecreaseAllowance operator (instead of Approve). To originally set an approval, use IncreaseAllowance, which works fine with no previous allowance. DecreaseAllowance is meant to be robust, that is if you decrease by more than the current allowance (eg. the user spent some in the middle), it will just round down to 0 and not make any underflow error.

## Messages

### IncreaseAllowance
Set or increase the allowance such that spender may access up to `current_allowance + amount` tokens from the env.sender account. This may optionally come with an Expiration time, which if set limits when the approval can be used (by time or height).

##### params
	spender
	amount
	expires

### DecreaseAllowance
Decrease or clear the allowance such that spender may access up to `current_allowance - amount` tokens from the env.sender account. This may optionally come with an Expiration time, which if set limits when the approval can be used (by time or height). If amount >= current_allowance, this will clear the allowance (delete it).

##### params
	spender
	amount
	expires

### TransferFrom
This makes use of an allowance and if there was a valid, un-expired pre-approval for the env.sender, then we move amount tokens from owner to recipient and deduct it from the available allowance.

##### params
	owner
	recipient
	amount

### SendFrom
SendFrom is to Send, what TransferFrom is to Transfer. 
This allows a pre-approved account to not just transfer the tokens, 
but to send them to another address to trigger a given action. 
Note SendFrom will set the Receive{sender} to be the env.sender 
(the account that triggered the transfer) rather than the owner account (the account the money is coming from). 
This is an open question whether we should switch this?

##### params
	owner
	recipient
	amount
	msg 

### BurnFrom
This works like TransferFrom, but burns the tokens instead of transfering them. 
This will reduce the owner's balance, total_supply and the caller's allowance.

##### params
	owner
	amount 

## Queries

### Allowance
This returns the available allowance that spender can access from the owner's account, along with the expiration info

TBD - how to keep this privacy preserving. Can't make this a message, otherwise it won't be useful to contracts. Setting a viewing key in a contract is possible, but that will make TXs that use this query super super expensive and very slow. Right now I feel like we'll have to allow it like this, and recommend that people use the _send_ and _receive_ functions instead 

##### params
	owner
	spender

##### Response
AllowanceResponse{balance, expiration}

# Mintable
This allows another contract to mint new tokens, possibly with a cap. 
There is only one minter specified here, if you want more complex access management, 
please use a multisig or other contract as the minter address and handle updating the ACL there.

## Messages
### Mint
If the env.sender is the allowed minter, this will create amount new tokens (updating total supply) and add them to the balance of recipient.
##### params
{recipient, amount}

## Queries
### Minter
Returns who and how much can be minted. Return type is MinterResponse {minter, cap}. Cap may be unset.
##### Params
  None

# Native

These are a type of Secret-20 coins which are backed by another _native_ asset. This is useful to create privacy tokens which wrap native assets, such as SCRT, ATOM, etc.

## Messages
### Deposit
Deposits a native coin into the contract, which will mint an equivalent amount of tokens to be created. Amount must be sent in the sent_funds field. 
The minted amounts MUST match the exchange rate specified by the ExchangeRate query. The deposit MUST return an error if any coins that do not match expected demoninations are sent.

##### params
  None

### Redeem
Redeems tokens in exchange for native coins. The redeemed tokens SHOULD be burned, and taken out of the pool

##### Params
	amount - Uint128. Number of tokens to redeem

	denom - string [optional] - *TBD* If a Secret-20 coin wishes to support multiple coins, this might be used to ask for currency in a specific denom. Though it would require specifying error conditions as well. We might want to remove supporting multiple native coins since it complicates things too much IMO

## Queries

### ExchangeRate
Gets information about the token exchange rate functionality that the contract provides.
This query MUST return: 
* exchange rate, in decimal form, the amount of native coins that equal one token.
* Denomination of native tokens which are acceptable, as a string OR a comma separated value

##### Params
  None
