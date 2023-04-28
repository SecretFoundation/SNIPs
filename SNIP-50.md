# SNIP-50 - Evaporation: Wasm Execution Gas Privacy

This feature is conceptually similar to the `padding` field; it allows clients to parameterize their message in order to mitigate data leaking through the publicly viewable execution metadata. Whereas `padding` is used to fill the length of encrypted messages, evaporation is used to fill the resultant `gas_used` field of execution transaction results.


## Rationale

The public `gas_used` result of a contract execution leaks data about the code path taken.

As a simple example, an attacker could use the `gas_used` information to distinguish between the following SNIP-2x execution methods with high confidence: create_viewing_key, set_viewing_key, increase_allowance/decrease_allowance, send/transfer, revoke_permit, and so on.

In some cases, an attacker could narrow or even deduce the range of possible values that certain arguments or storage items held during execution.

The proposed solution introduces a new API method, contract interface, and set of best practices. This approach prevents leaking `gas_used` to nodes as well since all evaporation takes place within the enclave.


## Evaporation

We introduce the concept of "evaporation", by which excess gas is deliberately and deterministically consumed during execution in order to pad the `gas_used`.

Users may include an optional `evaporate` field in every message. The value of the field should be an integer that specifies an arbitrary multiplier for some fixed-cost operation.

Using evaporation, wallets can compute a precise multiplier to provide as input during execution in order to produce a `gas_used` value that effectively obscures the nature of the transaction.


### Requests


| Name       | Type            | Description                                                                                                | optional |
|------------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
| gas_target | number          | The intended amount of gas to use for the execution.                                                       |  yes     |


#### Example

```json
{
  "transfer": {
    "recipient": "<address>",
    "amount": "100",
    "padding": "-------",
    "gas_target": 40000,
  },
}
```
