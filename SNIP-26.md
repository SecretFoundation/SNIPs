---
snip: SNIP-26
title: Allowances queries
author: Blake Regalia (@blake-regalia), Ben Adams (@darwinzer0)
status: Draft
type: Informational
created: 2023-04-28
requires: SNIP-20, SNIP-21, SNIP-22, SNIP-23, SNIP-24, SNIP-25
---

## Simple Summary

This document describes two new queries for SNIP-20 tokens: `AllowancesGiven` and `AllowancesReceived`. These queries give the ability for an owner to query for all allowances they have given out, and for a spender to query for all allowances they have received, respectively.

## Motivation

The idea has been discussed several times before but was met with hesitation due to the lack of an efficient implementation. This has been implemented in a manner that avoids any of the storage drawbacks brought up in prior discussion, which largely stemmed from how allowances were being stored in earlier reference implementations. 

https://github.com/SecretFoundation/SNIPs/pull/13
https://github.com/scrtlabs/snip20-reference-impl/pull/76
https://github.com/scrtlabs/snip20-reference-impl/issues/58

## Queries

### AllowancesGiven

This query MUST be authenticated.
 
Returns the list of allowances given out by the current account as an owner, as well as the total count of allowances given out.

Results SHOULD be paginated. Results MUST be sorted in reverse chronological order by the datetime at which the allowance was first created (i.e., order is not determined by expiration, nor by last modified).

##### Request

| Name | Type | Description | optional |
|------|------|-------------|----------|
| [with_permit].query.allowances_given.owner | string | Account from which tokens are allowed to be taken | no |
| [with_permit].query.allowances_given.page_size | number | Number of allowances to return, starting from the latest. i.e. n=1 will return only the latest allowance | no |
| [with_permit].query.allowances_given.page | number | Defaults to 0. Specifying a positive number will skip page * page_size txs from the start. | yes |

##### Response

```json
{
  "allowances_given": {
    "owner": "<address>",
    "allowances": [
      {
        "spender": "<address>",
        "allowance": "Uint128",
        "expiration": 1234,
      },
      { "...": "..." }
    ],
    "count": 200
  }
}
```

### AllowancesReceived

This query MUST be authenticated.

Returns the list of allowances given to the current account as a spender, as well as the total count of allowances received.

Results SHOULD be paginated. Results MUST be sorted in reverse chronological order by the datetime at which the allowance was first created (i.e., order is not determined by expiration).

##### Request

| Name | Type | Description | optional |
|------|------|-------------|----------|
| [with_permit.]query.allowances_received.spender | string | Account which is allowed to spend tokens on behalf of the _owner_ | no |
| [with_permit.]query.allowances_received.page_size | number | Number of allowances to return, starting from the latest. i.e. n=1 will return only the latest allowance | no |
| [with_permit.]query.allowances_received.page | number | Defaults to 0. Specifying a positive number will skip page * page_size txs from the start. | yes |

##### Response

```json
{
  "allowances_received": {
    "spender": "<address>",
    "allowances": [
      {
        "owner": "<address>",
        "allowance": "Uint128",
        "expiration": 1234,
      },
      { "...": "..." }
    ],
    "count": 200
  }
}
```

## Considerations

In order for pre-existing SNIP-20s to upgrade (once that feature is implemented in Secret Network), a CosmWasm `migrate` handler must be used since the allowances data struct is changed. 

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0).