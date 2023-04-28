---
snip: 50
title: Evaporation: Wasm Execution Gas Privacy
status: Draft
type: Informational
author: Blake Regalia (@blake-regalia), Ben Adams (@darwinzer0)
created: 2023-04-28
---

# SNIP-50 - Evaporation: Wasm Execution Gas Privacy

This feature is conceptually similar to the `padding` field; it allows clients to parameterize their message in order to mitigate data leaking through the publicly viewable execution metadata. Whereas `padding` is used to fill the length of encrypted messages, evaporation is used to fill the resultant `gas_used` field of execution transaction results.

## Rationale

The public `gas_used` result of a contract execution leaks data about the code path taken.

As a simple example, an attacker could use the `gas_used` information to distinguish between the following SNIP-2x execution methods with high confidence: create_viewing_key, set_viewing_key, increase_allowance/decrease_allowance, send/transfer, revoke_permit, and so on.

In some cases, an attacker could narrow or even deduce the range of possible values that certain arguments or storage items held during execution.

The proposed solution introduces two new API functions, contract interface, and set of best practices. This approach prevents leaking `gas_used` to nodes as well since all evaporation takes place within the enclave.

## Evaporation

We introduce the concept of "evaporation", by which excess gas is deliberately and deterministically consumed during execution in order to pad the `gas_used`.

Users may include an optional `evaporate` field in every message. The value of the field should be an integer that specifies an arbitrary multiplier for some fixed-cost operation.

Using evaporation, wallets can compute a precise target gas to provide as input during execution in order to produce a `gas_used` value that effectively obscures the nature of the transaction.

## API functions

#### `check_gas`

This API function returns the current amount of CosmWasm gas used as a `u64` and is called in a contract with `deps.api.check_gas()?`.

#### `evaporate_gas`

This API function burns a fixed amount of CosmWasm gas and is called in a contract with `deps.api.evaporate_gas(amount)?`, where `amount` is a `u32` value indicating the amount of gas to be used.

Together these API functions can be used to evaporate a calculated amount of left over gas at the end of the contract execution.

## Parameterized gas target approach

If we want to have an `ExecuteMsg` that uses an exact amount of gas that is set by a `gas_target` parameter you can do the following:

```rust
  ExecuteMsg::UseExact { gas_target } => {
    // other code added here

    let gas_used: u64 = deps.api.check_gas()?;

    let to_evaporate = gas_target - gas_used as u32;

    deps.api.gas_evaporate(to_evaporate)?;

    Ok(Response::default())
  }
```

Adding an optional `gas_target` parameter to all execute messages and using the above approach, any arbitrary gas usage can be targeted. The recommended approach for token contracts (SNIP-20s) is to add this optional parameter to all message types, so that wallets can set the target automatically.

Note, that there will be a small amount of wasm execution that occurs after the call to the `gas_evaporate` API function, therefore the exact amount of gas used can sometimes be off by 1 gas from the target using this strategy. Wallets can adjust the `gas_target` accordingly after testing. Alternatively, if the contract developer wishes, the gas_used can be fuzzied by adjusted it with a random number each time.

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

### Callback functions

The above approach works well for contract executions that do not send submessages. However, when using the receiver interface for SNIP-20 `send` messages and similar callbacks, developers might need to send additional gas target information through the `msg` parameter depending on the use case.

### Hard-coded gas target

Alternatively, if the gas used by a contract's function calls is deterministic, a contract developer can hard-code specific gas targets for execute messages. This approach is less flexible, however, and developers should make sure to add functionality to adjust the target values in case gas metering changes with a chain upgrade.