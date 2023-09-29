# SNIP-26 - Allowance queries for SNIP-20 tokens

This document describes two new queries for SNIP-20 tokens: `AllowancesGiven` and `AllowancesReceived`. These queries give the ability for an owner to query for all allowances they have given out, and for a spender to query for all allowances they have received, respectively.


## Queries

### AllowancesGiven

This query MUST be authenticated.
 
Returns the list of allowances given out by the current account as an owner, as well as the total count of allowances given out.

Results SHOULD be paginated. Results MUST be sorted in reverse chronological order by the datetime at which the allowance was first created (i.e., order is not determined by expiration, nor by last modified).

##### Request

| Name      | Type   | Description | optional |
|-----------|--------|-------------|----------|
| owner     | string | Account from which tokens are allowed to be taken | no |
| page_size | number | Number of allowances to return, starting from the latest. i.e. n=1 will return only the latest allowance | no |
| page      | number | Defaults to 0. Specifying a positive number will skip page * page_size txs from the start. | yes |

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

| Name      | Type   | Description | optional |
|-----------|--------|-------------|----------|
| spender   | string | Account which is allowed to spend tokens on behalf of the _owner_ | no |
| page_size | number | Number of allowances to return, starting from the latest. i.e. n=1 will return only the latest allowance | no |
| page      | number | Defaults to 0. Specifying a positive number will skip page * page_size txs from the start. | yes |

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

In order for pre-existing SNIP-20s to upgrade, a CosmWasm `migrate` handler must be used since the allowances data struct is changed. 
