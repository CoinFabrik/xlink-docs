# cross-peg-out-endpoint

- Location: `xlink/packages/contracts/bridge-stacks/contracts`
- Deployed contracts:[cross-peg-out-endpoint-v2-01](https://explorer.hiro.so/txid/SP2XD7417HGPRTREMKF748VNEQPDRR0RMANB7X1NK.cross-peg-out-endpoint-v2-01?chain=mainnet), [cross-peg-out-v2-01-agg](https://explorer.hiro.so/txid/SP673Z4BPB4R73359K9HE55F2X91V5BJTN5SXZ5T.cross-peg-out-v2-01-agg?chain=mainnet)

This technical document provides a detailed overview of the contracts responsible for managing the peg-out process, enabling the transfer of `SIP-010` bridged tokens from the Stacks network to EVM-compatible blockchain networks as EVM-based assets. The contracts manage token transfers by validating amounts and applying fees. During the process, tokens are either burned or transferred (depending on the token's configuration) to a designated address on the EVM chain.

The core functionalities of the module are implemented through public and governance functions in the following contracts:

- `cross-peg-out-endpoint-v2-01`: handles bridging of aBTC tokens from Stacks to EVM-compatible blockchains.
- `cross-peg-out-v2-01-agg`: enables the bridging of tokens from Stacks to an EVM-compatible blockchain to perform a swap on a non-ALEX liquidity aggregator.

## Storage

*All contracts include the following variables unless otherwise specified.*

### `is-paused`

| Data     | Type   |
| -------- | ------ |
| Variable | `bool` |

A flag that indicates whether the peg-out process is active. If set to `true`, all peg-out operations are paused, preventing any new transactions. The contract is deployed in a paused state by default.

### `use-whitelist`

*Only present in `cross-peg-out-endpoint-v2-01`.*

| Data     | Type   |
| -------- | ------ |
| Variable | `bool` |

A flag that determines whether a whitelist is enforced for peg-out operations. If set to `true`, only addresses explicitly whitelisted can execute peg-out transactions. By default, this feature is disabled.

### `whitelisted-users`

*Only present in `cross-peg-out-endpoint-v2-01`.*

| Data | Type               |
| ---- | ------------------ |
| Map  | `principal → bool` |

This map stores the whitelist status of users. Each entry associates a `principal` with a boolean value indicating whether they are authorized to perform peg-out transactions. When `use-whitelist` is set to `true`, users with a value of `true` in this map are whitelisted and permitted to participate in the process; otherwise, the whitelist is not enforced.

## Features

### Peg-out feature

#### `transfer-to-unwrap`

*In contract `cross-peg-out-endpoint-v2-01`.*

This function manages the peg-out process, enabling users to transfer `SIP-010` bridged tokens from the Stacks network to EVM-compatible blockchains. It begins by validating the transfer through the `validate-transfer-to-unwrap` function, performing checks such as token and chain approval, whitelist status, amount thresholds, and sufficient reserves for the operation.
Once validated, the function calculates the required fee and determines the net amount to be transferred. Depending on the token's properties, the function either burns the net amount directly from the user's balance or transfers the entire amount (including the fee) to the `cross-bridge-registry-v2-01`.
At the end of the process, the function logs key destination details, including the `settle-address`, which represents the recipient's address on the EVM-compatible blockchain.

##### Parameters

```lisp
(token-trait <ft-trait>)
(amount-in-fixed uint)
(dest-chain-id uint)
(settle-address (buff 256))
```

#### `transfer-to-swap`

*In contract `cross-peg-out-v2-01-agg`.*

This function handles the peg-out process from Stacks to an EVM-compatible blockchain. It prepares the transaction to be sent to an external liquidity aggregator in the `BridgeEndpointWithSwap.sol` contract. Its core functionality consists of validating amounts, deducting fees, transferring tokens and emitting an event with the transaction details.
The function begins by validating the transfer through the `validate-transfer-to-swap` function, performing checks such as token and chain approval, amount thresholds, and sufficient reserves for the operation.
Once validated, the function calculates the required fee and determines the net amount to be transferred. Depending on the token's properties, the function either burns the tokens or transfers them to the `cross-bridge-registry-v2-01` contract. This action records the transaction and prepares it for further processing.
At this point, an event is emitted, signaling relayers to detect and forward the transaction to `BridgeEndpointWithSwap`, which completes the operation through an external liquidity aggregator.

```lisp
(amount-in-fixed uint)
(token-in-trait <ft-trait>)
(token-out principal)
(min-amount-out (optional uint))
(dest-chain-id uint)
(settle-details { address: (buff 256), chain-id: (optional uint) })
```

### Governance features

#### `is-dao-or-extension`

This standard protocol function checks whether a caller (`tx-sender`) is the DAO executor or an authorized extension, delegating the extensions check to the `executor-dao` contract.

#### `is-whitelisted`

*Only present in `cross-peg-out-endpoint-v2-01`.*

A read-only function that checks whether a specific user is included in the `whitelisted-users` map. It returns `true` if the user is whitelisted; otherwise, it returns `false`.

##### Parameters

```lisp
(user principal)
```

#### `get-paused`

A read-only function that checks the operational status of the contract.

#### `set-paused`

A public function, governed through the `is-dao-or-extension`, that can change the contract's operational status.

##### Parameters

```lisp
(paused bool)
```

#### `apply-whitelist`

*Only present in `cross-peg-out-endpoint-v2-01`.*

A public function, governed through the `is-dao-or-extension`, that enables or disables the whitelist mechanism for peg-out transactions. When enabled, only users listed in the `whitelisted-users` map can perform peg-out operations.

##### Parameters

```lisp
(new-use-whitelist bool)
```

#### `whitelist`

*Only present in `cross-peg-out-endpoint-v2-01`.*

A public function, governed through the `is-dao-or-extension`, that allows the DAO to manage user access to the peg-out feature. It updates the `whitelisted-users` map to either include or remove a specific user. If the `whitelisted` parameter is set to `true`, the user is added to the whitelist; if set to `false`, the user is removed.

##### Parameters

```lisp
(user principal)
(whitelisted bool)
```

#### `whitelist-many`

*Only present in `cross-peg-out-endpoint-v2-01`.*

A public function, governed through the `is-dao-or-extension`, that allows the DAO to update the whitelist for multiple users in a single call. It works as a batch operation for the `whitelist` function, applying the specified whitelist status (`true` to add or `false` to remove) for each user in the provided list. The `whitelisted-users` map is updated.

##### Parameters

```lisp
(users (list 2000 principal))
(whitelisted (list 2000 bool))
```

### Getters

#### `get-use-whitelist`

*Only present in `cross-peg-out-endpoint-v2-01`.*

#### `get-approved-chain-or-fail`

##### Parameters

```lisp
(dest-chain-id uint)
```

#### `get-token-reserve-or-default`

##### Parameters

```lisp
(pair { token: principal, chain-id: uint })
```

#### `get-min-fee-or-default`

##### Parameters

```lisp
(pair { token: principal, chain-id: uint })
```

#### `get-approved-pair-or-fail`

##### Parameters

```lisp
(pair { token: principal, chain-id: uint })
```

## Contract calls (interactions)

- `executor-dao`: Calls are made to verify whether a certain contract-caller is designated as an extension.
- `cross-bridge-registry-v2-01`: This contract is called to verify key components of the peg-out process. It validates approved tokens and chain pairings, manages the deduction and queries of token reserves, and records accrued fees. It also handles updates to transaction statuses.
- `token-trait`: In the cross-peg-out process, this trait is employed to manage token operations such as burning tokens when required or transferring them to the `cross-bridge-registry-v2-01` contract. It is a customized version of Stacks' standard definition for Fungible Tokens (`sip-010`), with support for 8-digit fixed notation.

## Errors

| Error Name                     | Value         |
| ------------------------------ | ------------- |
| `ERR-NOT-AUTHORIZED`           | `(err u1000)` |
| `ERR-PAUSED`                   | `(err u1015)` |
| `ERR-USER-NOT-WHITELISTED`     | `(err u1016)` |
| `ERR-AMOUNT-LESS-THAN-MIN-FEE` | `(err u1017)` |
| `ERR-INVALID-AMOUNT`           | `(err u1019)` |

<!-- Documentation Contract Template v0.1.0 -->
