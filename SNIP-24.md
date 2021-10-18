# SNIP-24 - Query permits for [SNIP-20](/SNIP-20.md) tokens

This document describes a querying method for SNIP-20 tokens that is superior to using viewing keys. Contracts that support the existing SNIP-20 standard are still considered compliant, and clients should be written to benefit from these features when available, but provide fallback for when these features are not available.

The feature specified in this document is an improved UX for the `Allowance`, `Balance`, `TransferHistory` & `TransactionHistory` queries, aimed at enriching SNIP20 tokens usage and onset for new users.

- [SNIP-24 - Query permits for SNIP-20 tokens](#snip-24---query-permits-for-snip-20-tokens)
  - [Rationale](#rationale)
    - [The Problem with Viewing Keys](#the-problem-with-viewing-keys)
    - [Query Permits](#query-permits)
  - [Data Structures](#data-structures)
    - [`StdSignDoc`](#stdsigndoc)
      - [`PermitMsg`](#permitmsg)
      - [Full Example](#full-example)
    - [Signature](#signature)
  - [Messages](#messages)
    - [RevokePermit](#revokepermit)
      - [Request](#request)
  - [Queries](#queries)
    - [WithPermit](#withpermit)
      - [Allowance](#allowance)
      - [Balance](#balance)
      - [TransferHistory](#transferhistory)
      - [TransactionHistory](#transactionhistory)
  - [Usage examples](#usage-examples)
    - [Keplr](#keplr)
    - [secretcli](#secretcli)

## Rationale

### The Problem with Viewing Keys

Viewing keys are passwords meant to validate users at times when the blockchain cannot. Spesifically in queries, the query sender isn't authenticated and the contract doesn't know who is the querier. Therefore viewing keys were invented to provide a way of access control for users:

1. Alice sends a transaction `set_viewing_key(password)`
2. The contract stores `(alice,password)`
3. Later on, a query is sent to the contract `query("balance",alice,password)`
   - If `(alice,password)` matches what's in storage, the contract returns Alice's balance to the querier.

The main disadvantage of this method is that Alice must send a transaction before she can query her balance. This is bad UX for users new to Secret Network - Also because they have to pay SCRT gas to get a basic peice of information, but esspecialy when traffic is high and nodes lag behind, queried nodes might have the `query("balance")` answer but can't authnticate the querier because the nodes still didn't caught up with the `set_viewing_key()` transaction.

### Query Permits

Query permits are an alternative way of authenticating the querier. Instead of storing a password in the contract's state, the users signs a piece of data with their private keys, and then sends this data and the signature to the contract along with the query. The contrat then validates the signature against the data, and if it checks out returns an answer.

This way users don't have to send a transaction before they can access their data.

## Data Structures

The data structure for query permits was choosen to accomodate existing tools in the ecosystem, namley Keplr & secretcli which already know how to sign this and don't require extra code and support.

### `StdSignDoc`

The entire data being signed is a cosmos-sdk `StdSignDoc` with some constraints.

```go
// StdSignDoc is replay-prevention structure.
// It includes the result of msg.GetSignBytes(),
// as well as the ChainID (prevent cross chain replay)
// and the Sequence numbers for each signature (prevent
// inchain replay and enforce tx ordering per account).
type StdSignDoc struct {
	AccountNumber uint64            `json:"account_number" yaml:"account_number"`
	ChainID       string            `json:"chain_id" yaml:"chain_id"`
	Fee           json.RawMessage   `json:"fee" yaml:"fee"`
	Memo          string            `json:"memo" yaml:"memo"`
	Msgs          []json.RawMessage `json:"msgs" yaml:"msgs"`
	Sequence      uint64            `json:"sequence" yaml:"sequence"`
}
```

| Field           | Comment                                                             |
| --------------- | ------------------------------------------------------------------- |
| `AccountNumber` | must be `0`                                                         |
| `ChainID`       | freeform, but when signing with Keplr must use the current chain-id |
| `Fee`           | must be `0uscrt` with `1` gas                                       |
| `Memo`          | must be an empty string                                             |
| `Msgs`          | an array with only one message of the type `PermitMsg`              |
| `Sequence`      | must be `0`                                                         |

#### `PermitMsg`

`PermitMsg` is the only message allowed in `Msgs`.

```json
{
  "type": "query_permit",
  "value": {
    "permit_name": "freeform string",
    "allowed_tokens": ["<address_token_1>", "<address_token_2>"],
    "permissions": ["balance", "history", "allowance"]
  }
}
```

- `type` is always the string `query_permit`.
- `value.permit_name` is a freeform string. The users can later revoke this permit using this name.
- `value.allowed_tokens` is a list of token addresses to which this permit applies.
- `value.permissions` is on of `balance`, `history` or `allowance`.
  - `balance` - gives permission to query the `balance` of the permit signer.
  - `history` - gives permission to query the `transfer_history` and `transaction_history` of the permit signer.
  - `allowance` - gives permission to query the `allowance` of the permit signer as an `owner` ans as a `spender`.

#### Full Example

```json
{
  "chain_id": "secret-3",
  "account_number": "0",
  "sequence": "0",
  "msgs": [
    {
      "type": "query_permit",
      "value": {
        "permit_name": "test",
        "allowed_tokens": ["secret18vd8fpwxzck93qlwghaj6arh4p7c5n8978vsyg"],
        "permissions": ["balance"]
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

### Signature

```json

```

## Messages

### RevokePermit

A way for users to revoke permits that they signed in the past.

#### Request

| Name | Type   | Description            | optional |
| ---- | ------ | ---------------------- | -------- |
| name | string | The name of the permit | no       |

## Queries

### WithPermit

#### Allowance

#### Balance

#### TransferHistory

#### TransactionHistory

## Usage examples

### Keplr

```js
const permitName = "secretswap.io";
const allowedTokens = ["secret18vd8fpwxzck93qlwghaj6arh4p7c5n8978vsyg"];
const permissions = ["balance" /* , "history", "allowance" */];

const { signature } = await window.keplr.signAmino(
  chainId,
  myAddress,
  {
    chain_id: chainId,
    account_number: "0", // Must be 0
    sequence: "0", // Must be 0
    fee: {
      amount: [{ denom: "uscrt", amount: "0" }], // Must be 0 uscrt
      gas: "1", // Must be 1
    },
    msgs: [
      {
        type: "query_permit", // Must be "query_permit"
        value: {
          permit_name: permitName,
          allowed_tokens: allowedTokens,
          permissions: permissions,
        },
      },
    ],
    memo: "", // Must be empty
  },
  {
    preferNoSetFee: true,
    preferNoSetMemo: true,
  }
);

const { balance } = await secretjs.queryContractSmart(
  "secret18vd8fpwxzck93qlwghaj6arh4p7c5n8978vsyg",
  {
    with_permit: {
      query: { balance: {} },
      permit: {
        params: {
          permit_name: permitName,
          allowed_tokens: allowedTokens,
          chain_id: chainId,
          permissions: permissions,
        },
        signature: signature,
      },
    },
  }
);

console.log(balance.amount);
```

### secretcli

```console
$ echo '{
    "chain_id": "secret-3",
    "account_number": "0",
    "sequence": "0",
    "msgs": [
        {
            "type": "query_permit",
            "value": {
                "permit_name": "test",
                "allowed_tokens": [
                    "secret18vd8fpwxzck93qlwghaj6arh4p7c5n8978vsyg"
                ],
                "permissions": ["balance"],
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
}' > ./permit.json
$ secretcli tx sign-doc ./permit.json --from yo > ./sig.json
$ secretcli q compute query secret18vd8fpwxzck93qlwghaj6arh4p7c5n8978vsyg '{"with_permit":{"query":{"balance":{}},"permit":{"params":{"permit_name":"test","allowed_tokens":["secret18vd8fpwxzck93qlwghaj6arh4p7c5n8978vsyg"],"chain_id":"secret-3","permissions":["balance"]},"signature":'"$(cat ./sig.json)"'}}}'
```
