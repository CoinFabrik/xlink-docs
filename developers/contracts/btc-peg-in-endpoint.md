# btc-peg-in-endpoint

- Location: `xlink/packages/contracts/bridge-stacks/contracts`
- Deployed contracts: [btc-peg-in-endpoint-v2-05](https://explorer.hiro.so/txid/SP673Z4BPB4R73359K9HE55F2X91V5BJTN5SXZ5T.btc-peg-in-endpoint-v2-05?chain=mainnet), [btc-peg-in-endpoint-v2-05-lisa](https://explorer.hiro.so/txid/SP673Z4BPB4R73359K9HE55F2X91V5BJTN5SXZ5T.btc-peg-in-endpoint-v2-05-lisa?chain=mainnet), [btc-peg-in-v2-05-launchpad](https://explorer.hiro.so/txid/SP673Z4BPB4R73359K9HE55F2X91V5BJTN5SXZ5T.btc-peg-in-v2-05-launchpad?chain=mainnet), [btc-peg-in-v2-07-swap](https://explorer.hiro.so/txid/SP673Z4BPB4R73359K9HE55F2X91V5BJTN5SXZ5T.btc-peg-in-v2-07-swap?chain=mainnet), [btc-peg-in-v2-07a-agg](https://explorer.hiro.so/txid/SP673Z4BPB4R73359K9HE55F2X91V5BJTN5SXZ5T.btc-peg-in-v2-07a-agg?chain=mainnet)

This technical document provides a detailed overview of the contracts responsible for managing the peg-in process, enabling the transfer of BTC from the Bitcoin network to the Stacks network. In this process, BTC is represented as bridged tokens on Stacks (aBTC). The module's core functionality is implemented through a series of public functions distributed across five specialized contracts. Each contract addresses specific aspects of the BTC peg-in process.

This functionality is implemented and distributed across the following contracts:

- `btc-peg-in-endpoint-v2-05`: handles bridging BTC into the Stacks network, leveraging cross-router to manage the routing of BTC to the appropriate destination.
- `btc-peg-in-endpoint-v2-05-lisa`: extends Bitcoin peg-in operations by converting BTC into LiaBTC through intermediate bridging steps, ultimately enabling the issuance of BRC-20 tokens on Bitcoin.
- `btc-peg-in-v2-05-launchpad`: facilitates BTC peg-ins specifically for participation in launchpad projects on Stacks.
- `btc-peg-in-v2-07-swap`: enables the bridging of BTC into the Stacks network while enabling token swaps to convert BTC into other predefined assets during the process.
- `btc-peg-in-v2-07a-agg`: facilitates the BTC peg-in process for swaps with non-ALEX liquidity aggregators. Liquidity aggregators optimize token exchanges by accessing multiple liquidity sources. This allows ALEX to execute swaps even when there are no ALEX pools for a specific token pair.

## Storage

*All contracts include the following variables unless otherwise specified.*

### `fee-to-address`

| Data     | Type        |
| -------- | ----------- |
| Variable | `principal` |

The address where the fees collected from peg-in operations are transferred. By default, this address is set to the executor-dao responsible for governance.

### `peg-in-paused`

| Data     | Type   |
| -------- | ------ |
| Variable | `bool` |

A flag that indicates whether the peg-in process is paused. If set to `true`, all peg-in operations are suspended, preventing any new transactions. The contract is deployed in a paused state by default.

### `peg-in-fee`

| Data     | Type   |
| -------- | ------ |
| Variable | `uint` |

The percentage fee charged for peg-in transactions. By default, this value is `0`.

### `peg-in-min-fee`

| Data     | Type   |
| -------- | ------ |
| Variable | `uint` |

The minimum fee required for a peg-in transaction, regardless of the transaction amount. For each transaction, the fee is the maximum value between the calculated fee amount and the `peg-in-min-fee`. By default, this value is `0`.

### `btc-peg-out-fee`

*Only present in `btc-peg-in-v2-07-swap` and `btc-peg-in-v2-07a-agg`.*

| Data     | Type   |
| -------- | ------ |
| Variable | `uint` |

This variable represents the percentage fee applied to BTC peg-out operations during cross-swap transactions, where BTC is swapped and routed across chains to reach the final recipient. By default, it is initialized to `0` and can be updated via governance functions.

### `btc-peg-out-min-fee`

*Only present in `btc-peg-in-v2-07-swap` and `btc-peg-in-v2-07a-agg`.*

| Data     | Type   |
| -------- | ------ |
| Variable | `uint` |

This variable sets the minimum fee required for BTC peg-out operations during cross-swap transactions. By default, it is initialized to `0`.

## Features

### Public

#### `finalize-peg-in-cross`

*In contract `btc-peg-in-endpoint-v2-05`.*

This function manages the peg-in process for transferring BTC to Stacks with support for cross-chain routing. It validates the provided Bitcoin transaction (which represents the transfer of BTC to a peg-in address on the Bitcoin network), ensuring it has been mined and meets the necessary conditions including an approved peg-in address. The function also performs additional checks involving a "reveal transaction," which specifies the token and chain-id for the destination chain.
Once the transaction is validated, the function calculates fees, verifies that the asset being transferred is approved for bridging operations in the `.btc-bridge-registry-v2-01` contract, and registers the transaction status. Once validated, the function mints bridged BTC tokens and sends them to the recipient specified in the transaction details via the `.cross-router-v2-03`. If the validation or routing fails, a refund is executed.

##### Parameters

```lisp
(tx (buff 32768))
(block { header: (buff 80), height: uint })
(proof { tx-index: uint, hashes: (list 14 (buff 32)), tree-depth: uint })
(output-idx uint)
(reveal-tx { tx: (buff 32768), order-idx: uint })
(reveal-block { header: (buff 80), height: uint })
(reveal-proof { tx-index: uint, hashes: (list 14 (buff 32)), tree-depth: uint })
(token-out-trait <ft-trait>)
```

#### `finalize-peg-in-cross-swap`

*In contract `btc-peg-in-v2-07-swap`.*

This function mints bridged BTC tokens and swaps them according to routing instructions. Building upon the functionality of `finalize-peg-in-cross`, it adds token swapping capabilities during the peg-in process. In addition to the verifications performed in `finalize-peg-in-cross`, it ensures that routing configurations (e.g., minimum output amounts or token paths) are valid and meet the required conditions.
During the process, the function temporarily modifies the `peg-out-fee` and `peg-out-min-fee` values within the `btc-peg-out-endpoint-v2-01` contract. These adjustments allow the transaction to apply specific fee values during the routing and swap operations. Once the process is completed, the original values are restored.
It interacts with `.btc-bridge-registry-v2-01` contract to register transaction statuses. It also uses the `.cross-router-v2-03` to handle the cross operation and to perform token swaps through the `.amm-pool-v2-01` contract. The function mints bridged BTC tokens and swaps them according to routing instructions.

##### Parameters

```lisp
(tx (buff 32768))
(block { header: (buff 80), height: uint })
(proof { tx-index: uint, hashes: (list 14 (buff 32)), tree-depth: uint })
(output-idx uint)
(reveal-tx { tx: (buff 32768), order-idx: uint })
(routing-traits (list 5 <ft-trait>))
(token-out-trait <ft-trait>)
```

#### `finalize-peg-in-mint-liabtc`

*In contract `btc-peg-in-endpoint-v2-05-lisa`.*

The main purpose of this function is to convert BTC into LiaBTC, with the ultimate goal of issuing it as a BRC-20 token in Bitcoin. To achieve this, the function goes through an intermediate bridging step: after locking the desired amount of BTC on the Bitcoin network, it converts it into aBTC within the Stacks ecosystem. The aBTC is then wrapped into `wvLiaBTC`, a tokenized form of LiaBTC within Stacks.
Once the conversion is complete, the function initiates a peg-out operation using the `.meta-peg-out-endpoint-v2-04` contract. This step bridges the `wvLiaBTC` back to Bitcoin and issues it as BRC-20 tokens, sending it to a specific inscription address provided in the order. The function verifies both the validation of the Bitcoin transaction and the fulfillment of all LiaBTC minting conditions.
It also registers the peg-in transaction in the `.btc-bridge-registry-v2-01` contract. In case of any error, the refund mechanism is triggered to return the corresponding funds to the sender.

##### Parameters

```lisp
(tx (buff 32768))
(block { header: (buff 80), height: uint })
(proof { tx-index: uint, hashes: (list 14 (buff 32)), tree-depth: uint })
(output-idx uint)
(order-idx uint)
(message { token: principal, accrued-rewards: uint, update-block: uint })
(signature-packs (list 100
                        { signer: principal,
                        message-hash: (buff 32),
                        signature: (buff 65)
                        }))
```

#### `finalize-peg-in-launchpad`

*In contract `btc-peg-in-v2-05-launchpad`.*

This function is tailored for peg-ins associated with launchpad projects on Stacks. It uses `.btc-bridge-registry-v2-01` to validate and register transaction details while ensuring compatibility with the project’s parameters. These parameters include fields such as `user`, `launch-id`, and `payment-token-trait`, which define the specifics of the launchpad operation.
The function first mints bridged BTC tokens for the user and then registers the operation in the `.alex-launchpad-v2-03` contract. This registration involves transferring the minted bridged tokens assets to the launchpad contract on behalf of the user, where they are associated with the launchpad project.
In case of any error, it invokes the internal [refund](btc-peg-in-endpoint.md#relevant-internal-functions) function and logs the issue.

##### Parameters

```lisp
(tx (buff 32768))
(block { header: (buff 80), height: uint })
(proof { tx-index: uint, hashes: (list 14 (buff 32)), tree-depth: uint })
(output-idx uint)
(order-idx uint)
```

#### `finalize-peg-in-agg`

*In contract `btc-peg-in-v2-07a-agg`.*

This function facilitates a BTC peg-in operation designed for aggregated cross-chain routing. Unlike standard peg-in processes where users receive aBTC directly on Stacks, this function integrates a routing mechanism that forwards the bridged tokens for immediate cross-chain processing.
The process begins by verifying that the provided Bitcoin transaction has been mined and meets all peg-in validation criteria. It checks that the peg-in address is approved and calculates the required transaction fees. Once validated, the function mints aBTC for the net amount (after deducting fees) and registers the peg-in transaction in the `.btc-bridge-registry-v2-01` contract.
Instead of keeping the minted aBTC within the Stacks ecosystem, this function directly transfers it to the `.cross-peg-out-v2-01-agg` contract. This interaction enables automated routing and potential asset swaps to facilitate seamless movement of assets across blockchains. The function logs transaction details and ensures that any failure in the process triggers a refund mechanism.

##### Parameters

```lisp
(tx (buff 32768))
(block { header: (buff 80), height: uint })
(proof { tx-index: uint, hashes: (list 14 (buff 32)), tree-depth: uint })
(output-idx uint)
(reveal-tx { tx: (buff 32768), order-idx: uint })
(reveal-block { header: (buff 80), height: uint })
(reveal-proof { tx-index: uint, hashes: (list 14 (buff 32)), tree-depth: uint })
```

### Governance features

#### `is-dao-or-extension`

This standard protocol function checks whether a caller (`tx-sender`) is the DAO executor or an authorized extension, delegating the extensions check to the `executor-dao` contract.

#### `is-peg-in-paused`

A read-only function that checks the operational status of the contract.

#### `pause-peg-in`

A public function, governed through the `is-dao-or-extension`, that can change the contract's operational status.

##### Parameters

```lisp
(paused bool)
```

#### `set-fee-to-address`

This feature establishes the address to which the fees collected from peg-in operations are transferred.

##### Parameters

```lisp
(new-fee-to-address principal)
```

#### `set-peg-in-fee`

This feature sets the fee for the peg-in operation as a percentage of the transaction amount.

##### Parameters

```lisp
(fee uint)
```

#### `set-peg-in-min-fee`

This feature allows to set the minimum fee required for a peg-in transaction.

##### Parameters

```lisp
(fee uint)
```

#### `set-btc-peg-out-fee`

*Only present in `btc-peg-in-v2-07-swap` and `btc-peg-in-v2-07a-agg`.*

This feature sets the percentage fee applied to BTC peg-out operations.

##### Parameters

```lisp
(fee uint)
```

#### `set-btc-peg-out-min-fee`

*Only present in `btc-peg-in-v2-07-swap` and `btc-peg-in-v2-07a-agg`.*

This feature establishes the minimum fee required for BTC peg-out operations.

##### Parameters

```lisp
(fee uint)
```

### Supporting features

The following functions are tools to assist the off-chain activities.

1. Construct and destruct helpers (`destruct-principal`, `construct-principal`).
2. Order creation helpers (`create-order-cross-or-fail`, `create-order-cross-swap-or-fail`, `create-order-mint-liabtc-or-fail`, `create-order-launchpad-or-fail`, `create-order-agg-or-fail`).
3. Decoding helpers (`decode-order-cross-from-reveal-tx-or-fail`, `decode-order-cross-swap-from-reveal-tx-or-fail`, `decode-order-agg-or-fail`, `decode-order-agg-from-reveal-tx-or-fail`).

### Relevant internal functions

- `refund`: this function returns the total amount of the peg-in transaction, including the fee, to the sender in case of a failure during the operation.

### Getters

#### `get-peg-in-fee`

#### `get-peg-in-min-fee`

#### `get-fee-to-address`

#### `get-peg-in-sent-or-default`

##### Parameters

```lisp
(tx (buff 32768))
(output uint)
```

#### `get-txid`

##### Parameters

```lisp
(tx (buff 32768))
```

#### `get-btc-peg-out-fee` *Only present in `btc-peg-in-v2-07-swap` and `btc-peg-in-v2-07a-agg`.*

#### `get-btc-peg-out-min-fee` *Only present in `btc-peg-in-v2-07-swap` and `btc-peg-in-v2-07a-agg`.*

#### `get-liabtc-decimals` *Only present in `btc-peg-in-endpoint-v2-05-lisa`.*

## Contract calls (interactions)

- `executor-dao`: calls are made to verify whether a certain contract-caller is designated as an extension.
- `btc-bridge-registry-v2-01`: this contract is called to validate peg-in addresses, check and set transaction statuses, and order details during peg-in operations.
- `bridge-common-v2-02`: this contract is called to extract and validate Bitcoin transaction details, decode routing and order information during peg-in operations.
- `token-abtc`: this contract handles the management of aBTC (Bridged BTC) tokens, representing BTC on the Stacks network. It is called to mint and transfer aBTC during peg-in operations.
- `cross-router-v2-03`: this contract is called to route tokens and execute cross-chain transfers during advanced peg-in operations, such as cross and cross-swap transactions.
- `alex-launchpad-v2-03`: this contract is called to register and validate peg-in operations associated with launchpad projects on the Stacks network.
- `clarity-bitcoin-v1-07`: this contract is called to retrieve the transaction ID of native SegWit Bitcoin transactions by excluding witness data.
- `btc-peg-out-endpoint-v2-01`: this contract is called to manage refunds during peg-in failures to transfer BTC back to users.
- `liabtc-mint-endpoint`: this contract is called to validate and mint liabtc during peg-in operations.
- `token-wvliabtc`: this contract handles the management of wvliabtc tokens, the wrapped representation of liabtc on the Stacks network. It is called to mint and manage wvliabtc during peg-in operations.
- `meta-peg-out-endpoint-v2-04`: this contract is called to bridge wvLiaBTC from the Stacks network to Bitcoin as a BRC-20 token, transferring it to a specified address.
- `cross-peg-out-v2-01-agg`: this contract is called to finalize the transaction by preparing the swap to be executed in an EVM-compatible blockchain.

## Errors

| Error Name                     | Value         |
| ------------------------------ | ------------- |
| `err-unauthorised`             | `(err u1000)` |
| `err-paused`                   | `(err u1001)` |
| `err-peg-in-address-not-found` | `(err u1002)` |
| `err-invalid-amount`           | `(err u1003)` |
| `err-invalid-tx`               | `(err u1004)` |
| `err-already-sent`             | `(err u1005)` |
| `err-bitcoin-tx-not-mined`     | `(err u1011)` |
| `err-invalid-input`            | `(err u1012)` |
| `err-token-mismatch`           | `(err u1015)` |
| `err-slippage`                 | `(err u1016)` |
| `err-not-in-whitelist`         | `(err u1017)` |
| `err-invalid-routing`          | `(err u1018)` |
| `err-commit-tx-mismatch`       | `(err u1019)` |
| `err-invalid-token`            | `(err u1020)` |

<!-- Documentation Contract Template v0.1.0 -->
