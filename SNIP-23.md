# SNIP-24 - Improvements to Allowance in [SNIP-20](/SNIP-20.md)

This document describes several optional enhancements to the SNIP-20 spec.
Contracts that support the existing SNIP-20 standard are still considered
compliant, and clients should be written so as to benefit from these features
when available, but provide fallbacks for when these features are not available.

The features specified in this document are mostly miscellaneous quality-of-life
improvements, aimed at enriching UI integration, and making auditability of
accounts easier (provided a viewing key, of course!).

* [IncreaseAllowanceEx and DecreaseAllowanceEx](#increaseallowanceex-and-decreaseallowanceex)
    * [IncreaseAllowanceEx](#increaseallowanceex)
    * [DecreaseAllowanceEx](#decreaseallowanceex)
* [Available allowance](#available-allowance)
* [AllowanceEx query](#allowanceex-query)

## IncreaseAllowanceEx and DecreaseAllowanceEx
The existing `IncreaseAllowance` and `DecreaseAllowance` messages allow
optionally specifying a unix time at which the allowance is no longer valid.
When giving allowance to another account for the first time, if the `expiration`
field was not specified, then the expiration defaulted to `"never"`. On subsequent
calls to these messages, not specifying `expiration` would keep the existing
expiration setting unchanged. The problem was that after setting an expiration
time, there was no way to set it to `"never"` again. The closest option was
setting the unix time to `u64::MAX`.

This SNIP defines the [`Expiration`](./SNIP-721.md#expiration) type as defined in SNIP-721:

* `"never"` - the approval will never expire
* `{"at_time": 1700000000}` - the approval will expire 1700000000 seconds after 01/01/1970 (time value is u64)
* `{"at_height": 3000000}` - the approval will expire at blockheight 3000000 (height value is u64)

... and proposes adding two new queries:

### IncreaseAllowanceEx
Set or increase the allowance such that `spender` may access up to
`current_allowance + amount` tokens from the Cosmos message sender account.
This may optionally come with an expiration parameter, which if set limits when the
approval can be used (by time). If the expiration is not provided, it should default
to `"never"` if no previous allowance is still valid, and on subsequent calls 
should not change the defined expiration.

##### Request
|Name         |Type             |Description                                                                                                   | optional |
|-------------|-----------------|--------------------------------------------------------------------------------------------------------------|----------|
|spender      | string          |  The address of the account getting access to the funds                                                      | no       |
|amount       | string(Uint128) |  The number of tokens to increase allowance by                                                               | no       |
|expiration   | Expiration      |  Time or block height at which the allowance expires.                                                        | yes      |
|padding      | string          |  Ignored string used to maintain constant-length messages                                                    | yes      |

##### Response
```json
{
  "increase_allowance_ex": {
    "spender": "<address>",
    "owner": "<address>",
    "allowance": "<current allowance>"
  }
}
```


### DecreaseAllowanceEx
Set or increase the allowance such that `spender` may access up to
`current_allowance + amount` tokens from the Cosmos message sender account.
If amount is equal to or greater than the current allowance, this action MUST
set the allowance to zero, and return a `"success"` response.
This may optionally come with an expiration parameter, which if set limits when the
approval can be used (by time). If the expiration is not provided, it should default
to `"never"` if no previous allowance is still valid, and on subsequent calls
should not change the defined expiration.

##### Request
|Name         |Type             |Description                                                                                                   | optional |
|-------------|-----------------|--------------------------------------------------------------------------------------------------------------|----------|
|spender      | string          |  The address of the account getting access to the funds                                                      | no       |
|amount       | string(Uint128) |  The number of tokens to decrease allowance by                                                               | no       |
|expiration   | Expiration      |  Time or block height at which the allowance expires.                                                        | yes      |
|padding      | string          |  Ignored string used to maintain constant-length messages                                                    | yes      |

##### Response
```json
{
  "decrease_allowance_ex": {
    "spender": "<address>",
    "owner": "<address>",
    "allowance": "<current allowance>"
  }
}
```

## Available allowance
The allowance given by an account may be much bigger than the actual balance
of that account.
This means that contracts can not trust that the allowance that was granted
to them by users is backed by liquid funds.
To solve this, `Allowance` (as well as `AllowanceEx` defined below)
MAY now return a new optional field called `available` which is equal to
`min(allowance, user_balance)`. This will allow users and contracts to know
how much of the allowance is currently available for spending, without exposing
the balance of the user if it is much bigger than the granted allowance.

The available balance should be 0 if the allowance is expired. Queries don't
currently have knowledge of the block height and time, but contracts can record
the block parameters of the last block in which they were executed,
[as described in SNIP-721](./SNIP-721.md#queryblockinfo).

NOTE: The expiration field in the response should be `0` (zero) if
`{In,De}creaseAlloanceEx` set the expiration to `"at_height"`, and should
be `null` or not included if the expiration was set to `"never"`.

For example, if user A granted user B allowance for 5K TOKEN, then user B will
only be able to know user A's balance if it is lower than the allowance.

Example SNIP-21 `Allowance` query response:
```json
{
  "allowance": {
    "spender": "secret1xyz",
    "owner": "secret1xyz",
    "allowance": "2000",
    "available": "500",
    "expiration": 100000
  }
}
```

## AllowanceEx query
This new query is identical to [`Allowance`](./SNIP-20.md#allowance) but returns
a full [`Expiration`](./SNIP-721.md#expiration) object in the `expiration`
field. This query is defined separately from `Allowance` because there's no
backwards compatible way of adding this more detailed information to the
existing query.
