# SNIP-21 - Minor improvements to [SNIP-20](/SNIP-20.md)

This document describes several optional enhancements to the SNIP-20 spec.
Contracts that support the existing SNIP-20 standard are still considered
compliant, and clients should be written so as to benefit from these features
when available, but provide fallbacks for when these features are not available.

The features specified in this document are mostly miscellaneous quality-of-life
improvements, aimed at enriching UI integration, and making auditability of
accounts easier (provided a viewing key, of course!).

* [Private Memo](#Private-Memo)
* [Transfer History](#Transfer-History)
    * [Total Transfer Count](#Total-Transfer-Count)
    * [Block Time and Height](#Block-Time-and-Height)
* [Transaction History](#Transaction-History)

## Private Memo

The existing `Transfer`, `TransferFrom`, `Send`, `SendFrom`,
`Burn`, `Mint`, and `BurnFrom` messages MAY
also accept a `memo` field, which is just a regular JSON string. This memo
SHOULD be saved in the transaction history of the balances involved in the
transaction (`from`, `sender`, and `receiver`) and served as the `memo` field
of items in the responses to the `TransferHistory` and `TransactionHistory`
(introduced in this document) queries.

Example `Transfer` message:
```json
{
  "recipient": "secret1xyz",
  "amount": "123000000",
  "memo": "A private message only the sender and recipient can see"
}
```

Example `TransferHistory` response:
```json
{
  "transfer_history": {
    "txs": [
      {
        "id": 123,
        "from": "secret1xyz",
        "sender": "secret1xyz",
        "receiver": "secret1xyz",
        "coins": {
          "denom": "FOOBAR",
          "amount": "123000000"
        },
        "memo": "Second private message"
      },
      {
        "id": 123,
        "from": "secret1xyz",
        "sender": "secret1xyz",
        "receiver": "secret1xyz",
        "coins": {
          "denom": "FOOBAR",
          "amount": "123000000"
        },
        "memo": "First private message"
      }
    ]
  }
}
```

### Receiver Interface
When sending a `Receive` callback as part of a `Send` or `SendFrom` transaction,
The `Receive` message SHOULD contain the same `memo` that was supplied in the
original message.

## Transfer History

### Total Transfer Count

The `TransferHistory` query MAY return an additional field `total` which is a
64 bit JSON integer representing the total count of transfer events registered
in the history of the balance being queried.

The absence of this feature makes presenting good paging interfaces very hard
for user interfaces. There's no way of knowing how many pages there are in
the history without first fetching all the registered transfers and seeing when
we get an empty page. With this feature, if the UI wants to display 10 transfers
at a time, the first query to `TransferHistory` will return the 10 latest
transfers, and the UI will know it can offer `(total / 10) + 1` pages that can
be jumped to.

Example `TransferHistory` response:
```json
{
  "transfer_history": {
    "total": 200,
    "txs": [
      {
        "id": 123,
        "from": "secret1xyz",
        "sender": "secret1xyz",
        "receiver": "secret1xyz",
        "coins": {
          "denom": "FOOBAR",
          "amount": "123000000"
        }
      },
      {
        "id": 123,
        "from": "secret1xyz",
        "sender": "secret1xyz",
        "receiver": "secret1xyz",
        "coins": {
          "denom": "FOOBAR",
          "amount": "123000000"
        }
      }
    ]
  }
}
```

### Block Time and Height

The items returned by the `TransferHistory` query MAY contain additional fields
called `block_time` and `block_height`. These two fields MUST correspond to the
block time and block height of the block that included the transaction that
recorded the transfer being reported. The block time is in Unix time.

Example `TransferHistory` response:
```json
{
  "transfer_history": {
    "txs": [
      {
        "id": 123,
        "from": "secret1xyz",
        "sender": "secret1xyz",
        "receiver": "secret1xyz",
        "coins": {
          "denom": "FOOBAR",
          "amount": "123000000"
        },
        "block_time": 12006,
        "block_height": 101
      },
      {
        "id": 123,
        "from": "secret1xyz",
        "sender": "secret1xyz",
        "receiver": "secret1xyz",
        "coins": {
          "denom": "FOOBAR",
          "amount": "123000000"
        },
        "block_time": 12000,
        "block_height": 100
      }
    ]
  }
}
```

## Transaction History

The original `TransferHistory` query paints an incomplete picture of users'
interactions with the token. It only records operations performed through
`Transfer`, `TransferFrom`, `Send`, and `SendFrom`, while the other operations
which move funds around - `Mint`, `Burn`, `Deposit`, `Withdraw` etc. are not
recorded and can not be easily tracked by users after the fact.

The following is the specification of the new `TransactionHistory` query, which
aims to provide a complete description of the history of users' balances.

---

This query MUST be authenticated.

This query SHOULD return a list of json objects describing the changes to the
balance of the querying address, in newest-first order. The user may optionally
specify a limit on the amount of information returned by paging the available
items.

The response should look as described below.
* The `total` field is REQUIRED (unlike as in `TransferHistory` where it is
  missing for legacy reasons) and must equal to the number of recorded
  transactions related to the querying account.
* The `id` field is optional, and may be used by contracts that choose
  to populate it.
* The `memo` field may be missing or `null`, but if present it MUST be the
  memo that was attached to the message sent by the user in the described
  transaction.
* The `block_time` and `block_height` MUST be the same as the fields of the same
  name described above in
  [Block Time and Height](#Block-Time-and-Height).

More details are described below.

##### Request
|Name         |Type             |Description                                                                                                  | optional |
|-------------|-----------------|-------------------------------------------------------------------------------------------------------------|----------|
| key         | string          | The viewing key                                                                                             | no       |
| address     | string          | Addresses SHOULD be a valid bech32 address, but contracts may use a different naming scheme as well         | no       |
| page_size   | number          | Number of transactions to return, starting from the latest. i.e. n=1 will return only the latest transaction| no       |
| page        | number          | This defaults to 0. Specifying a positive number will skip `page * page_size` txs from the start.           | yes      |


##### Response
The `action` field of each transaction under `txs` is an object, which is one of
the variants of `TxAction`.
```json
{
  "transaction_history": {
    "total": 200,
    "txs": [
      {
        "id": "optional ID",
        "block_time": 12000,
        "block_height": 100,
        "coins": {
          "denom": "coin denomination/name",
          "amount": "Uint128"
        },
        "memo": "private message",
        "action": {}
      },
      { "...": "..." }
    ]
  }
}
```

`TxAction` has the following variants:

* `Transfer` - represents transactions triggered by `Transfer`, `TransferFrom`,
  `Send`, and `SendFrom`. The fields in this object correspond to the fields
  described in `TransferHistory`.
  * `from` - The address from which funds were moved.
  * `sender` - The address which performed the operation.
  * `recipient` - The address to which funds were moved.
  
  `from` and `sender` will differ in events created by the `*From` messages,
  which use allowance from a third `from` address to perform the operation.
  
```json
{
  "transfer": {
    "from": "secret1xyz",
    "sender": "secret1xyz",
    "recipient": "secret1xyz"
  }
}
```

* `Mint` - represents transactions triggered by `Mint`. The `minter` field
  shows the address of the minter which created the tokens, and the `recipient`
  field shows the address of the account that received the tokens. This item
  will show up in both the minter's and recipient's histories.

```json
{
  "mint": {
    "minter": "secret1xyz",
    "recipient": "secret1xyz"
  }
}
```

* `Burn` - represents transactions triggered by `Burn`.
  * `burner` - The address that performed the burn.
  * `owner` - The address whose funds were burned.
  
  This item will show up in both the burner's and owner's histories.
  `burner` and `owner` will differ in events created by `BurnFrom`,

```json
{
  "burn": {
    "burner": "secret1xyz",
    "owner": "secret1xyz"
  }
}
```

* `Deposit` - represents a `Deposit` operation, where native tokens were
  deposited to the contract for the address of the querier.
  
```json
{
  "deposit": {}
}
```

* `Redeem` - represents a `Redeem` operation, where native tokens were withdrawn
  from the account of the address of the querier.
  
```json
{
  "redeem": {}
}
```
