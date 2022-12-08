# Transaction Fees <a id="transaction-fees"></a>
{% hint style="success" %}
NOTE: This document is written based on the latest EVM which have been used since the `Kore` hardfork.
If you want the previous document, please refer to [previous document](transaction-fees-previous.md).

`Kore` hardfork block number is as follows.
* Baobab Testnet: `will be updated`
* Cypress Mainnet: `will be updated`
{% endhint %}

The transaction fee of one transaction is calculated as follows:

```text
(Transaction Fee) := (Gas Used) * (Base Fee)
```

* The `Gas used` is computed by adding the next two gas costs;
  * The intrinsic gas cost calculated based on the account type and transaction type. I will explain it here.
  * If the transaction is contract executing type, gas cost calculated on KLVM during the contract execution. For more information, please refer [klvm docs](../computation/klaytn-virtual-machine/klaytn-virtual-machine-previous.md).
* `Base Fee` is the actual gas price used for the transaction. It has the same meaning as the `Effective Gas Price`.

This calculated transaction fee is subtracted from the sender's or fee payer's account balance, depending on the transaction.

## Gas and Base Fee Overview <a id="gas-and-base-fee-overview"></a>
### Gas <a id="gas"></a>
Every action that changes the state of the blockchain requires gas. When a node processes a user's transaction, such as sending KLAY, using KIP-7 tokens, or executing a contract, the user has to pay for the computation and storage usage. The payment amount is decided by the amount of `gas` required.

`Gas` is a measuring unit representing how much calculation is needed to process the user's transaction.

### Dynamic Gas Fee Mechanism <a id="dynamic-gas-fee-mechanism"></a>
Since the Klaytn v1.9.0 hard fork, a dynamic gas fee mechanism has replaced the existing fixed fee policy. Dynamic gas fee policy provides a stable service to users by preventing network abuse and storage overuse. The gas fee changes according to the network situation. Seven parameters affect the `base fee(gas fee)`:

1. PREVIOUS_BASE_FEE: Base fee of the previous block
2. GAS_USED_FOR_THE_PREVIOUS_BLOCK: Gas used to process all transactions of the previous block
3. GAS_TARGET: The gas amount that determines the increase or decrease of the base fee (30 million at the moment)
4. MAX_BLOCK_GAS_USED_FOR_BASE_FEE: Implicit block gas limit to enforce the max basefee change rate (60 million at the moment)
5. BASE_FEE_DELTA_REDUCING_DENOMINATOR: The value to set the maximum base fee change to 5% per block (20 at the moment, can be changed later by governance)
6. UPPER_BOUND_BASE_FEE: The maximum value for the base fee (750 ston at the moment, can be changed later by governance)
7. LOWER_BOUND_BASE_FEE: The minimum value for the base fee (25 ston at the moment, can be changed later by governance)

### Base Fee <a id="base-fee"></a>
The basic idea of this algorithm is that the `base fee` would go up if the gas used exceeds the base gas and vice versa. It is closely related to the number of transactions in the network and the gas used in the process. There is an upper and lower limit for the `base fee` to prevent the fee from increasing or decreasing indefinitely. There is also a cap for the gas and an adjustment value for the fluctuation to prevent abrupt changes in the `base fee`. The values can be changed by governance.

```text
(BASE_FEE_CHANGE_RATE) = (GAS_USED_FOR_THE_PREVIOUS_BLOCK - GAS_TARGET)
(ADJUSTED_BASE_FEE_CHANGE_RATE) = (BASE_FEE_CHANGE_RATE) / (GAS_TARGET) / (BASE_FEE_DELTA_REDUCING_DENOMINATOR)
(BASE_FEE_CHANGE_RANGE) = (PREVIOUS_BASE_FEE) * (ADJUSTED_BASE_FEE_CHANGE_RATE)
(BASE_FEE) = (PREVIOUS_BASE_FEE) + (BASE_FEE_CHANGE_RANGE) 
```

The `base fee` is calculated for every block; there could be changes every second. Transactions from a single block use the same `base fee` to calculate transaction fees. Only transactions with a gas price higher than the block `base fee` can be included in the block. Half of the transaction fee for each block is burned (BURN_RATIO = 0.5, cannot be changed by governance).

> NOTE: An important feature that sets Klaytn apart from Ethereum's EIP-1559 is that it does not have tips. Klaytn follows the First Come, First Served(FCFS) principle for its transactions.

### Transaction Replacement <a id="transaction-replacement"></a>

Klaytn currently does not provide a way to replace a transaction using the unit price but may support different methods for the transaction replacement in the future. Note that in Ethereum, a transaction with a given nonce can be replaced by a new one with a higher gas price.

## Intrinsic Gas Cost  <a id="intrinsic-gas-cost"></a>

Basically, Klaytn is keeping compatibility with Ethereum. So Klaytn's gas cost calcuation is pretty similar with that of Ethereum. However, due to the unique features, there are several new gas costs for those features.

| TxType | Gas |
| :--- | :--- |
| LegacyTransaction | TxGas + PayloadGas + KeyValidationGas |
| ValueTransfer | TxGasValueTransfer + KeyValidationGas |
| ValueTransferMemo | TxGasValueTransfer + PayloadGas + KeyValidationGas |
| AccountUpdate | TxGasAccountUpdate + KeyCreationGas + KeyValidationGas |
| SmartContractDeploy | TxGasContractCreation + PayloadGas + KeyValidationGas |
| SmartContractExecution | TxGasContractExecution + PayloadGas + KeyValidationGas |
| Cancel | TxGasCancel + KeyValidationGas |

### 1. Gas Formula for Transaction Types <a id="gas-formula-for-transaction-types"></a>
| Item | Gas | Description |
| :--- | :--- | :--- |
| TxGasValueTransfer | 21000 | Gas required to transfer KLAY |
| TxGasContractCreation | 53000 | Amount of gas paid for contract-creating transactions |
| TxGasContractExecution | 21000 | Base gas for contract execution |

### 2. Gas Formula for PayloadGas <a id="gas-formula-for-payloadgas"></a>
| Item | Gas | Description |
| :--- | :--- | :--- |
| TxDataZeroGas | 4 | Amount of gas paid for every zero byte of data or code for a legacy-typed transaction |
| TxDataNonZeroGas | 68 | Amount of gas paid for every nonzero byte of data or code for a legacy-typed transaction |
| TxDataGas | 100 | Gas required per transaction's single byte for a non-legacy-typed transaction |

Gas for payload data is calculated as below

```text
GasPayload = number_of_bytes * TxDataGas
```

### 3. Gas Formula for Key <a id="gas-formula-for-key"></a>

| Item | Gas | Description |
| :--- | :--- | :--- |
| TxAccountCreationGasPerKey | 20000 | Gas required for a key-pair creation |
| TxValidationGasPerKey | 15000 | Gas required for a key validation |
| TxGasAccountUpdate | 21000 | Gas required for an account update |
| TxGasFeeDelegated | 10000 | Gas required for a fee delegation |
| TxGasFeeDelegatedWithRatio | 15000 | Gas required for a fee delegation with ratio |
| TxGasCancel | 21000 | Gas required to cancel a transaction which has a same nonce |

KeyValidationGas is defined as below based on the key type,

| Key Type | Gas |
| :--- | :--- |
| Nil | N/A |
| Legacy | 0 |
| Fail | 0 |
| Public | 0 |
| MultiSig | \(number of signatures - 1\) \* GasValidationPerKey \(15000\) |
| RoleBased | Based on keys in the role used in the validation |

KeyCreationGas is defined as below based on the key type,

| Key Type | Gas |
| :--- | :--- |
| Nil | N/A |
| Legacy | 0 |
| Fail | 0 |
| Public | GasCreationPerKey \(20000\) |
| MultiSig | \(keys\) \* GasCreationPerKey |
| RoleBased | Gas fee calculated based on keys in each role. e.g., GasRoleTransaction = \(keys\) _GasCreationPerKey_ _GasRoleAccountUpdate = \(keys\)_ GasCreationPerKey GasRoleFeePayer = \(keys\) \* GasCreationPerKey |

