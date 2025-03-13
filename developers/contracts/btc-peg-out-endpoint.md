# btc-peg-out-endpoint-v2-01.clar

- Location: `xlink/packages/contracts/bridge-stacks/contracts/btc-peg-out-endpoint-v2-01.clar`
- [Deployed contract](https://explorer.hiro.so/txid/SP2XD7417HGPRTREMKF748VNEQPDRR0RMANB7X1NK.btc-peg-out-endpoint-v2-01?chain=mainnet)

This technical document provides a detailed overview of the contract responsible for managing the peg-out process, enabling the transfer of bridged BTC from the Stacks network back to the Bitcoin network. In this process, aBTC (Bridged BTC tokens on Stacks) is burned or transferred, depending on the context, and BTC is released to a specified Bitcoin address. The contract's primary functionality is implemented through a series of public functions. Let's review this core operation.

## Storage

### `fee-to-address`

| Data     | Type        |
| -------- | ----------- |
| Variable | `principal` |

The address where the fees collected from peg-out operations are transferred. By default, the address assigned to receive these fees is the `tx-sender` address of the contract deployer.

### `peg-out-paused`

| Data     | Type   |
| -------- | ------ |
| Variable | `bool` |

A flag that indicates whether the peg-out process is active. If set to true, all peg-out operations are paused, preventing any new transactions. The contract is deployed in a paused state by default.

### `peg-out-fee`

| Data     | Type   |
| -------- | ------ |
| Variable | `uint` |

The percentage fee charged for peg-out transactions. The fee is represented in a fixed-point format with 6 decimal places. This means that 100% is represented as `u100000000`and 1% is represented as `u1000000`. By default, this value is `u0`.

### `peg-out-min-fee`

| Data     | Type   |
| -------- | ------ |
| Variable | `uint` |

The minimum fee required for a peg-out transaction, regardless of the transaction amount. By default, this value is `u0`.

## Features

#### `request-peg-out-0`

###### _(in contract btc-peg-out-endpoint-v2-01)_

This function initiates the peg-out process, enabling users to transfer bridged BTC from the Stacks network to a specified Bitcoin address (peg-out-address). First, it validates the requested amount using `validate-peg-out-0` to ensure it meets the minimum requirements (the requested amount must be sufficient to cover both the peg-out fee and the gas fee, leaving a positive net amount available for transfer). The function registers the request by interacting with the `.btc-bridge-registry-v2-01` contract, storing details such as the requesting user, target Bitcoin address, calculated fees, and associated block heights (e.g., `block-height` and `burn-block-height`). A unique request ID is generated for tracking. Finally, the amount of aBTC is escrowed by transferring it from the user to the contract, securing the funds until the request is either completed or revoked.

##### Parameters

```lisp
(peg-out-address (buff 128))
(amount uint)
```

#### `claim-peg-out`

###### _(in contract btc-peg-out-endpoint-v2-01)_

This function allows a user to claim a specific peg-out request. The function first retrieves the request details and performs several validations to check that the request is active and valid. It checks that the peg-out process is not paused, the request has not been finalized, revoked, or already claimed, and that all conditions for claiming are met. Upon successful validation, the function registers the request as claimed in the `.btc-bridge-registry-v2-01` contract, updating the state with the claimer's identity, the Bitcoin address (`fulfilled-by`) responsible for completing the transaction, and the block height defining the claim's expiration period. Note that the expiration period defines a specific timeframe during which the claimed request must be finalized. And finally, the updated request details are logged, and the claimer is granted the right to proceed with finalizing the peg-out.

##### Parameters

```lisp
(request-id uint)
(fulfilled-by (buff 128))
```

#### `finalize-peg-out`

###### _(in contract btc-peg-out-endpoint-v2-01)_

This function completes the peg-out process. It performs checks to finalize the peg-out, including verifying that the transaction has been mined, and verifies that the included details in the transaction (concerning the amount, recipient address, and fulfiller address) align with the request specifications. The function interacts with the `.btc-bridge-registry-v2-01` contract to mark the request as finalized.
Additionally, the function processes fees by transferring them to the designated governance address. It then handles the bridged tokens (aBTC) associated with the request in one of two ways:

1. Approved Peg-in Address:
   - The tokens are burned, reducing the total supply.
2. Third-Party Address:
   - The tokens are transferred from the contract to the claimer, maintaining the total supply.

Once all operations are completed, the function logs the transaction details and confirms the successful finalization of the request.

##### Parameters

```lisp
(request-id uint)
(tx (buff 32768))
(block { header: (buff 80), height: uint })
(proof { tx-index: uint, hashes: (list 14 (buff 32)), tree-depth: uint })
(output-idx uint)
(fulfilled-by-idx uint)
```

#### `revoke-peg-out`

###### _(in contract btc-peg-out-endpoint-v2-01)_

This function allows the user who created a peg-out request to cancel it, only if certain conditions are met. The function checks that the required `request-revoke-grace-period` has passed since the request was created, and verifies that the request has not already been claimed, finalized, or already revoked. Once these validations are passed, the function updates the request's status to "revoked" in the `.btc-bridge-registry-v2-01` contract. It then processes the refund by transferring the associated fees and the bridged tokens (aBTC) back to the requester.

##### Parameters

```lisp
(request-id uint)
```

### Governance features

#### `is-dao-or-extension`

This standard protocol function checks whether a caller (`tx-sender`) is the DAO executor or an authorized extension, delegating the extensions check to the `executor-dao` contract.

#### `is-peg-out-paused`

A read-only function that checks the operational status of the contract.

#### `pause-peg-out`

A public function, governed through the `is-dao-or-extension`, that can change the contract's operational status.

##### Parameters

```lisp
(paused bool)
```

### Getters

#### `get-peg-out-fee`

#### `get-peg-out-min-fee`

#### `get-request-revoke-grace-period`

#### `get-request-claim-grace-period`

#### `get-request-or-fail`

##### Parameters

```lisp
(request-id uint)
```

#### `get-peg-in-sent-or-default`

##### Parameters

```lisp
(tx (buff 32768))
(output uint)
```

#### `get-fee-to-address`

#### `get-txid`

##### Parameters

```lisp
(tx (buff 32768))
```

## Contract calls (interactions)

- `executor-dao`: Calls are made to verify whether a certain contract-caller is designated as an extension.
- `btc-bridge-registry-v2-01`: This contract is called to register, update, and track peg-out requests.
- `clarity-bitcoin-v1-07`: This contract is called to validate Bitcoin transactions by verifying that they have been mined and extracting relevant transaction details, such as inputs, outputs, and script data.
- `token-abtc`: This contract represents the aBTC token on the Stacks network. It is directly responsible for managing the transfer, burning, and refunding of aBTC tokens during the peg-out process.

## Errors

| Error Name                      | Value         |
| ------------------------------- | ------------- |
| `err-unauthorised`              | `(err u1000)` |
| `err-paused`                    | `(err u1001)` |
| `err-invalid-amount`            | `(err u1003)` |
| `err-invalid-tx`                | `(err u1004)` |
| `err-already-sent`              | `(err u1005)` |
| `err-address-mismatch`          | `(err u1006)` |
| `err-request-already-revoked`   | `(err u1007)` |
| `err-request-already-finalized` | `(err u1008)` |
| `err-revoke-grace-period`       | `(err u1009)` |
| `err-request-already-claimed`   | `(err u1010)` |
| `err-bitcoin-tx-not-mined`      | `(err u1011)` |
| `err-tx-mined-before-request`   | `(err u1013)` |
| `err-slippage`                  | `(err u1016)` |

<!-- Documentation Contract Template v0.1.0 -->
