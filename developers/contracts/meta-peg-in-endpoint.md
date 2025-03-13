# meta-peg-in-endpoint

- Location: `xlink/packages/contracts/bridge-stacks/contracts`
- Deployed contracts: [meta-peg-in-endpoint-v2-04](https://explorer.hiro.so/txid/SP673Z4BPB4R73359K9HE55F2X91V5BJTN5SXZ5T.meta-peg-in-endpoint-v2-04?chain=mainnet), [meta-peg-in-endpoint-v2-04-lisa](https://explorer.hiro.so/txid/SP673Z4BPB4R73359K9HE55F2X91V5BJTN5SXZ5T.meta-peg-in-endpoint-v2-04-lisa?chain=mainnet), [meta-peg-in-v2-06-swap](https://explorer.hiro.so/txid/SP673Z4BPB4R73359K9HE55F2X91V5BJTN5SXZ5T.meta-peg-in-v2-06-swap?chain=mainnet).

This technical document provides a comprehensive overview on the module responsible for the peg-in (bridging) of Bitcoin's BRC-20 assets to the Stacks network. BRC-20 is a token standard on Bitcoin's metaprotocol layer, enabling fungible assets similar to Ethereum's ERC-20. The module's core functionality is implemented through a series of public functions distributed across multiple specialized contracts, each addressing specific aspects of the peg-in process.

This functionality is implemented and distributed across the following contracts:

- `meta-peg-in-endpoint-v2-04`: facilitates the bridging of BRC-20 tokens from Bitcoin to Stacks, leveraging cross-router for routing to the appropriate destination.
- `meta-peg-in-endpoint-v2-04-lisa`: facilitates peg-in operations for BRC-20 tokens from Bitcoin that involve burning LiaBTC tokens within the ALEX system.
- `meta-peg-in-v2-06-swap`: facilitates peg-in operations for BRC-20 tokens from Bitcoin to Stacks with integrated asset swapping capabilities.

## Storage

###### _(all contracts include the following variables unless otherwise specified)_

### `paused`

| Data     | Type   |
| -------- | ------ |
| Variable | `bool` |

This data variable serves as a flag to control the operational status of the contract. When set to `true`, all meta peg-in transactions are suspended. By default, the contract is deployed in _paused_ mode. Refer to the [pause](meta-peg-in-endpoint.md#pause) governance feature to know more on how to change this status.

### `fee-to-address`

| Data     | Type        |
| -------- | ----------- |
| Variable | `principal` |

This is the address where fees, charged in Bridged BTC tokens (token-abtc), are sent. By default, this address is the `tx-sender` who deployed the contract.

### `peg-in-fee`

| Data     | Type   |
| -------- | ------ |
| Variable | `uint` |

This value represents the fee required for peg-in operations, expressed in fixed BTC (Bitcoin's base unit: satoshis). Fee amounts below this limit will be rejected with the error `err-invalid-amount`. By default, this value is set to `0`.

### `btc-peg-out-fee`

###### _(only present in meta-peg-in-v2-06-swap)_

| Data     | Type   |
| -------- | ------ |
| Variable | `uint` |

This variable represents the percentage fee applied to BTC peg-out operations during cross-swap transactions. By default, it is initialized to `u0` and can be updated via governance functions. Refer to the [set-btc-peg-out-fee](meta-peg-in-endpoint.md#set-btc-peg-out-fee) governance feature to know more about updating this variable.

### `btc-peg-out-min-fee`

###### _(only present in meta-peg-in-v2-06-swap)_

| Data     | Type   |
| -------- | ------ |
| Variable | `uint` |

This variable sets the minimum fee required for BTC peg-out operations during cross-swap transactions. By default, it is initialized to `u0` and can be updated via governance functions. Refer to the [set-btc-peg-out-min-fee](meta-peg-in-endpoint.md#set-btc-peg-out-min-fee) governance feature to know more about updating this variable.

### Relevant constants

#### `burn-height-start`

| Type   | Value                                                                                         |
| ------ | --------------------------------------------------------------------------------------------- |
| `uint` | `burn-block-height` [(see more)](https://docs.stacks.co/reference/keywords#burn-block-height) |

This constant denotes the block height at which the contract was deployed on the underlying burn blockchain. It serves to restrict operations to only include transactions minted from this block onward.

## Features

#### `finalize-peg-in-cross-on-index`

###### _(in contract meta-peg-in-endpoint-v2-04)_

This function processes the order recipient, which is extracted and decoded from the provided reveal transaction. The reveal transaction contains additional details about the peg-in operation, such as the recipient's address and the `order-idx`. In this scenario, the recipient may be on a different blockchain. To facilitate the peg-in process, the contract calls the supporting contract `cross-router-v2-03`, to route the tokens based on the destination chain.
As this peg-in feature is designed for Bitcoin metaprotocol tokens (e.g., BRC-20), the primary objective of this function is to facilitate 'crossing' to an EVM network. In such cases, the `cross-router-v2-03` contract will subsequently call `cross-peg-out-endpoint-v2-01` to execute the transfer of BRC-20 tokens from Bitcoin to an EVM chain. If any errors occur during the cross-transaction data validation, the function triggers a refund mechanism.

##### Parameters

```lisp
(tx {
    bitcoin-tx: (buff 32768),
    output: uint,
    tick: (string-utf8 256),
    amt: uint
    , from: (buff 128),
    to: (buff 128),
    from-bal: uint,
    to-bal: uint,
    decimals: uint
    })
(block { header: (buff 80), height: uint })
(proof { tx-index: uint, hashes: (list 14 (buff 32)), tree-depth: uint })
(signature-packs (list 10 {
                            signer: principal,
                            tx-hash: (buff 32),
                            signature: (buff 65)
                            }))
(reveal-tx { tx: (buff 32768), order-idx: uint })
(reveal-block { header: (buff 80), height: uint })
(reveal-proof { tx-index: uint, hashes: (list 14 (buff 32)), tree-depth: uint })
(fee-idx (optional uint))
(token-trait <ft-trait>)
(token-out-trait <ft-trait>)
```

#### `finalize-peg-in-cross-swap-on-index`

###### _(in contract meta-peg-in-v2-06-swap)_

Similar to the peg-in-cross, the peg-in-cross-swap involves a cross-blockchain operation with an additional asset swapping capability. This feature is useful when the desired target token requires intermediate swap operations. The function achieves this by receiving a list of token traits (interfaces that define the behavior of tokens in the operation) involved in the operation up to the target token. Refer to the [AMM Pool documentation](https://docs.alexgo.io/developers/protocol-contracts#amm-trading-pool) for more details. In addition to the sender (from) and recipient (to) addresses, the order details in the reveal transaction include information about the swap route. This swap route is validated against the `amm-pool-v2-01` contract through the `cross-router-v2-03` contract to check pool parameters. Additionally, this function includes a refund mechanism that gets triggered in case any errors occur during the cross-swap transaction validation.

##### Parameters

```lisp
(tx {
      bitcoin-tx: (buff 32768),
      output: uint,
      tick: (string-utf8 256),
      amt: uint,
      from: (buff 128),
      to: (buff 128),
      from-bal: uint,
      to-bal: uint,
      decimals: uint
    })
(block { header: (buff 80), height: uint })
(proof { tx-index: uint, hashes: (list 14 (buff 32)), tree-depth: uint })
(signature-packs (list 10 { signer: principal, tx-hash: (buff 32), signature: (buff 65) }))
(reveal-tx { tx: (buff 32768), order-idx: uint })
(reveal-block { header: (buff 80), height: uint })
(reveal-proof { tx-index: uint, hashes: (list 14 (buff 32)), tree-depth: uint })
(fee-idx (optional uint))
(routing-traits (list 5 <ft-trait>))
(token-out-trait <ft-trait>)
```

#### `finalize-peg-in-request-burn-liabtc-on-index`

###### _(in contract meta-peg-in-endpoint-v2-04-lisa)_

This function handles a peg-in operation that involves requesting the burn of LiaBTC tokens in the ALEX system. It begins by indexing the provided Bitcoin transaction using the `oracle-v2-01` contract to verify its validity and ensure it has not been processed before. Once the transaction is validated, the function calls `finalize-peg-in-request-burn-liabtc` to complete the burn request.
The function interacts with the `meta-bridge-registry-v2-03-lisa` contract to retrieve and validate token pair details, which refer to the registered combination of a token and its target chain ID, and with the `liabtc-mint-endpoint` to register the burn request. Additionally, it utilizes the `clarity-bitcoin-v1-07` contract to extract transaction details such as the SegWit transaction ID. Any accrued rewards or updates, which are likely tied to the use or staking of LiaBTC tokens within the ALEX system, are processed through the provided `liabtc-message` and validated using signature packs.
The function includes mechanisms to handle fees, using the `btc-bridge-registry-v2-01` contract to register these transactions and to ensure the peg-in-fee is properly accounted for.

##### Parameters

```lisp
(tx {
    bitcoin-tx: (buff 32768),
    output: uint,
    tick: (string-utf8 256),
    amt: uint,
    from: (buff 128),
    to: (buff 128),
    from-bal: uint,
    to-bal: uint,
    decimals: uint
    })
(block { header: (buff 80), height: uint })
(proof { tx-index: uint, hashes: (list 14 (buff 32)), tree-depth: uint })
(signature-packs (list 10 { signer: principal, tx-hash: (buff 32), signature: (buff 65) }))
(reveal-tx { tx: (buff 32768), order-idx: uint })
(reveal-block { header: (buff 80), height: uint })
(reveal-proof { tx-index: uint, hashes: (list 14 (buff 32)), tree-depth: uint })
(fee-idx (optional uint))
(liabtc-message { token: principal, accrued-rewards: uint, update-block: uint })
(liabtc-signature-packs (list 100 {
                                signer: principal,
                                message-hash: (buff 32),
                                signature: (buff 65)
                                }))
```

#### `finalize-peg-in-update-burn-liabtc`

###### _(in contract meta-peg-in-endpoint-v2-04-lisa)_

This function finalizes or revokes a burn request for LiaBTC tokens initiated in the ALEX system. It validates the Bitcoin transaction through the `oracle-v2-01` contract, ensuring it was mined and indexed correctly. It interacts with the `meta-bridge-registry-v2-03-lisa` contract to verify the status of the burn request and update its details.
If the burn request is finalized (status `0x01`), the function confirms the burn through the `liabtc-mint-endpoint` and completes the operation. For revocations (status `0x02`), the function handles re-minting of LiaBTC tokens and executes a peg-out process using the `meta-peg-out-endpoint-v2-04`.

##### Parameters

```lisp
(commit-tx { tx: (buff 32768), fee-idx: (optional uint) })
(block { header: (buff 80), height: uint })
(proof { tx-index: uint, hashes: (list 14 (buff 32)), tree-depth: uint })
(reveal-tx { tx: (buff 32768), order-idx: uint })
(reveal-block { header: (buff 80), height: uint })
(reveal-proof { tx-index: uint, hashes: (list 14 (buff 32)), tree-depth: uint })
(message { token: principal, accrued-rewards: uint, update-block: uint })
(signature-packs (list 100 {
                            signer: principal,
                            message-hash: (buff 32),
                            signature: (buff 65)
                            }))
```

### Supporting features

The following functions are tools to assist the off-chain activities.

1. Validation helpers (`validate-tx-cross`, `validate-tx-cross-swap`, `validate-tx-request-burn`, `validate-tx-update-burn`).
2. Order creation helpers (`create-order-cross-or-fail`, `create-order-cross-swap-or-fail`, `create-order-request-burn-or-fail`).
3. Decoding helpers (`decode-order-cross-or-fail`, `decode-order-cross-swap-or-fail`).
4. Token pair helpers (`get-pair-details-many`, `is-approved-pair`).

### Governance features

#### `is-dao-or-extension`

This standard protocol function checks whether a caller (`tx-sender`) is the DAO executor or an authorized extension, delegating the extensions check to the `executor-dao` contract.

#### `is-paused`

A read-only function that checks the operational status of the contract.

#### `pause`

A public function, governed through the `is-dao-or-extension`, that can change the contract's operational status.

##### Parameters

```lisp
(new-paused bool)
```

#### `transfer-all-to-many`

This contract feature is designed to transfer its entire balance of a specified list of tokens to a new owner. Access to this feature is restricted by the `is-dao-or-extension` function.

##### Parameters

```lisp
(new-owner principal) (token-traits (list 10 <ft-trait>))
```

#### `set-fee-to-address`

A public function, governed through the `is-dao-or-extension`, that sets the address to which the fee in Bridged BTC tokens will be transferred.

##### Parameters

```lisp
(new-fee-to-address principal)
```

#### `set-peg-in-fee`

A public function, governed through the `is-dao-or-extension`, that establishes the fee to be deducted for the peg-in operation.

##### Parameters

```lisp
(fee uint)
```

#### `set-btc-peg-out-fee`

###### _(only present in meta-peg-in-v2-06-swap)_

A public function, governed through the `is-dao-or-extension`, that sets the percentage fee applied to BTC peg-out operations.

##### Parameters

```lisp
(fee uint)
```

#### `set-btc-peg-out-min-fee`

###### _(only present in meta-peg-in-v2-06-swap)_

A public function, governed through the `is-dao-or-extension`, that establishes the minimum fee required for BTC peg-out operations.

##### Parameters

```lisp
(fee uint)
```

### Getters

- `get-fee-to-address`
- `get-peg-in-fee`
- `get-pair-details`
- `get-pair-details-or-fail`
- `get-pair-details-many`
- `get-tick-to-pair-or-fail`
- `get-peg-in-sent-or-default`
- `get-btc-peg-out-fee` _(only present in meta-peg-in-v2-06-swap)_
- `get-btc-peg-out-min-fee` _(only present in meta-peg-in-v2-06-swap)_
- `get-liabtc-decimals` _(only present in meta-peg-in-endpoint-v2-04-lisa)_

### Relevant internal functions

- `index-tx`: this function validates and indexes a Bitcoin transaction in the `oracle-v2-01` contract.
- `refund`: this function returns the total amount of the peg-in transaction, including the fee, to the sender in case of a failure during the operation.

## Contract calls (interactions)

- `executor-dao`: calls are made to verify whether a certain contract-caller is designated as an extension.
- `meta-bridge-registry-v2-03`: this interaction is used by the peg-in contract to check and set token pair information. It also facilitates liquidity operations to set balances and manage pool operations.
- `btc-peg-out-endpoint-v2-01`: the `request-peg-out-0` function of this contract is used by the peg-in contract to remove liquidity and refund BTC amounts.
- `btc-bridge-registry-v2-01`: the peg-in contract calls the `set-peg-in-sent` function of this contract to register a processed Bitcoin transaction.
- `meta-peg-out-endpoint-v2-04`: the `finalize-peg-in-remove-liquidity` feature and the internal `refund` function call this contract to request a peg-out in their respective contexts.
- `cross-router-v2-03`: the peg-in cross and peg-in cross swap operations use this contract to validate-route and route peg-ins involving other blockchains.
- `oracle-v2-01`: this contract is called to verify that a transaction was mined and to index Bitcoin transactions.
- `clarity-bitcoin-v1-07`: the peg-in contract uses this contract to obtain legacy transaction IDs from a SegWit native transaction.
- `amm-pool-v2-01`: this contract manages all liquidity operations, including adding and reducing positions, within the peg-in features. Additionally, it acts as a helper to retrieve pool information and details.
- `token-abtc`: this is the Bridged BTC token used throughout the peg-in contract to operate with native Stacks' SIP-010 token transactions involving the burn blockchain base-coin (BTC).
- Tokens (`token-trait`): during peg-in operations, this trait is used to invoke the relevant tokens to perform the necessary transfers involved in the transaction. It is a customized version of Stacks' standard definition for Fungible Tokens (`sip-010`), with support for 8-digit fixed notation.
- `bridge-common-v2-02`: this contract provides common helper functions to all xLink contracts, including order creation, transaction decoding, and cross-chain routing validations.
- `liabtc-mint-endpoint`: this contract is called to validate and register burn requests of LiaBTC during peg-in operations. It ensures that the burn process adheres to protocol requirements and updates the registry with the burn request details.
- `token-wvliabtc`: this contract manages the wrapped representation of LiaBTC tokens on the Stacks network. It is used during peg-in operations to burn wvliabtc tokens and to validate the corresponding token amounts for bridging purposes.

## Errors

| Error Name                                                                | Value         |
| ------------------------------------------------------------------------- | ------------- |
| `err-unauthorised`                                                        | `(err u1000)` |
| `err-paused`                                                              | `(err u1001)` |
| `err-peg-in-address-not-found`                                            | `(err u1002)` |
| `err-invalid-amount`                                                      | `(err u1003)` |
| `err-token-mismatch`                                                      | `(err u1004)` |
| `err-invalid-tx`                                                          | `(err u1005)` |
| `err-already-sent`                                                        | `(err u1006)` |
| `err-address-mismatch`                                                    | `(err u1007)` |
| `err-request-already-revoked`                                             | `(err u1008)` |
| `err-request-already-finalized`                                           | `(err u1009)` |
| `err-revoke-grace-period`                                                 | `(err u1010)` |
| `err-request-already-claimed`                                             | `(err u1011)` |
| `err-invalid-input`                                                       | `(err u1012)` |
| `err-tx-mined-before-request`                                             | `(err u1013)` |
| `err-commit-tx-mismatch`                                                  | `(err u1014)` |
| `err-invalid-burn-height`                                                 | `(err u1003)` |
| `err-tx-mined-before-start`                                               | `(err u1015)` |
| `err-slippage-error`                                                      | `(err u1016)` |
| `err-bitcoin-tx-not-mined`                                                | `(err u1017)` |
| `err-invalid-request` _(only present in meta-peg-in-endpoint-v2-04-lisa)_ | `(err u1018)` |

<!-- Documentation Contract Template v0.1.0 -->
