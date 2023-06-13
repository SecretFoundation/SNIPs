# SNIP-25 - Improved privacy for [SNIP-20](/SNIP-20.md) tokens

This document describes enhancements to the SNIP-20 spec, namely, the addition of `decoys` and `entropy` in execution messages in order to mitigate storage access pattern attacks. Decoys are an optional reference to a vector of Addr which, when provided, add an additional layer of privacy by obfuscating the real accounts that are being updated. Execution messages which accept the `decoy` field also accept the `entropy` field, which is an optional Binary used to introduce randomness by providing additional read and writes to the decoy addresses that are included in the message.

Contracts that support the existing SNIP-20 standard are still considered
compliant, and clients should be written so as to benefit from this feature
when available.

- [Decoys and Entropy](#Decoys-And-Entropy)

## Decoys and Entropy

The existing `Redeem`, `Deposit`, `Transfer`, `Send`, `Burn`, `TransferFrom`, `SendFrom`, `BurnFrom`, and `Mint` messages now may also accept `decoy` and `entropy` fields.

Example `Transfer` message, with optional reference to `decoys` and `entropy`:

##### Request

```json
{
  "transfer": {
    "recipient": "secret1jxlkaddg4nvsvnjylx94xnwrjprqvvp20r4hxd",
    "amount": "1000000000000000000",
    "memo": "Transfer for services rendered",
    "decoys": [
      "secret1zpaxzv6yggvmj4jj7md94rjxvnpa6q07asxyyh",
      "secret12g3ujhpkunstpdg78f8fny0r59zg6e9kghgpxy"
    ],
    "account_random_pos":
    //account_random_pos is the position of the real account in the decoys array
  }
}
```

##### Response

```json
{
  "transfer": {
    "status": "success"
  }
}
```

### Decoys + Entropy Implementation

The [`update_balance` function](https://github.com/scrtlabs/snip20-reference-impl/blob/ea9fb0cd76f3e0d48e86b4d02a3990f2f4a84d00/src/state.rs#L136) makes use of `decoys` and `entropy`.

If `decoys` is None, it loads the current balance of the account, updates it (either adds or subtracts the amount_to_be_updated), checks if the new balance is valid (in case of subtraction, the balance must not be negative), and saves the new balance back to the storage.

If `decoys` is not None, it creates a new vector `accounts_to_be_written` which includes the account to be updated and all `decoy` accounts, ordered as they are in the `decoys_vec`. For each account in this vector, it loads the current balance from the storage. If the current account is the real one to be updated and it has not been updated before, the balance is updated. The (possibly updated) balance is then saved back to storage.

```rust
pub fn update_balance(
        store: &mut dyn Storage,
        account: &Addr,
        amount_to_be_updated: u128,
        should_add: bool,
        operation_name: &str,
        decoys: &Option<Vec<Addr>>,
        account_random_pos: &Option<usize>,
    ) -> StdResult<()> {
        match decoys {
            None => {
                let mut balance = Self::load(store, account);
                balance = match should_add {
                    true => {
                        safe_add(&mut balance, amount_to_be_updated);
                        balance
                    }
                    false => {
                        if let Some(balance) = balance.checked_sub(amount_to_be_updated) {
                            balance
                        } else {
                            return Err(StdError::generic_err(format!(
                                "insufficient funds to {operation_name}: balance={balance}, required={amount_to_be_updated}",
                            )));
                        }
                    }
                };

                Self::save(store, account, balance)
            }
            Some(decoys_vec) => {
                // It should always be set when decoys_vec is set
                let account_pos = account_random_pos.unwrap();

                let mut accounts_to_be_written: Vec<&Addr> = vec![];

                let (first_part, second_part) = decoys_vec.split_at(account_pos);
                accounts_to_be_written.extend(first_part);
                accounts_to_be_written.push(account);
                accounts_to_be_written.extend(second_part);

                // In a case where the account is also a decoy somehow
                let mut was_account_updated = false;

                for acc in accounts_to_be_written.iter() {
                    // Always load account balance to obfuscate the real account
                    // Please note that decoys are not always present in the DB. In this case it is ok beacuse load will return 0.
                    let mut acc_balance = Self::load(store, acc);
                    let mut new_balance = acc_balance;

                    if *acc == account && !was_account_updated {
                        was_account_updated = true;
                        new_balance = match should_add {
                            true => {
                                safe_add(&mut acc_balance, amount_to_be_updated);
                                acc_balance
                            }
                            false => {
                                if let Some(balance) = acc_balance.checked_sub(amount_to_be_updated)
                                {
                                    balance
                                } else {
                                    return Err(StdError::generic_err(format!(
                                        "insufficient funds to {operation_name}: balance={acc_balance}, required={amount_to_be_updated}",
                                    )));
                                }
                            }
                        };
                    }
                    Self::save(store, acc, new_balance)?;
                }

                Ok(())
            }
        }
    }
}
```
