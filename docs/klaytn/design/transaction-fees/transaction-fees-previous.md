# Transaction Fees <a id="transaction-fees"></a>

{% hint style="success" %}
NOTE: This document is written based on the initial EVM used before the activation of the protocol upgrade.
If you want the latest document, please refer to [latest document](transaction-fees.md).
{% endhint %}

The transaction fee of one transaction is computed as follows:

```text
Transaction fee := (Gas used) x (unit price)
```

* The `Gas used` is computed by adding the next two gas costs;
  * The intrinsic gas cost calculated based on the transaction type. I will explain it here.
  * If the transaction is contract executing type, gas cost calculated on KLVM during the contract execution. For more information, please refer [klvm docs](../computation/klaytn-virtual-machine/klaytn-virtual-machine-previous.md).
* `unit price` is the price of gas defined in Klaytn.

This calculated transaction fee is subtracted from the sender's or fee payer's balance, depending on the transaction.

## Gas and Unit Price Overview <a id="gas-and-unit-price-overview"></a>
### Gas <a id="gas"></a>

Every action that changes the state of the blockchain requires gas. When a node processes user's transaction such as sending KLAY, using ERC-20 tokens, or executing a contract, the user has to pay for the computation and storage usage. The amount of payment is decided by the amount of `gas` required.

`Gas` is a measuring unit representing how much calculation is needed to process the user's transaction.

### Unit Price <a id="unit-price"></a>

`Unit price` is the price for a single gas. The unit price \(also called `gas price`\) is set in the system by the governance. It is currently set to 250 ston \(_i.e._, 250 x 10^9 peb\) per gas and cannot be changed by user. The current value of the unit price can be obtained by calling the `klay.gasPrice` API.

In Ethereum, users set the gas price for each transaction, and miners choose which transactions to be included in their block to maximize their reward. It is something like bidding for limited resources. This approach has been working because it is market-based. However, the transaction cost fluctuates and often becomes too high to guarantee the execution.

To solve the problem, Klaytn is using a fixed unit price and the price can be adjusted by the governance council. This policy ensures that every transaction will be handled equally and be guaranteed to be executed. Therefore, users do not need to struggle to determine the right unit price.

#### Transaction Validation against Unit Price <a id="transaction-validation-against-unit-price"></a>

Klaytn only accepts transactions with gas prices, which can be set by the user, that are equal to the unit price of Klaytn; it rejects transactions with gas prices that are different from the unit price in Klaytn.

#### Unit Price Error <a id="unit-price-error"></a>

The error message `invalid unit price` is returned when the gas price of a transaction is not equal to the unit price of Klaytn.

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
# legacy-typed transaction
PayloadGas = number_of_zero_bytes x G_txdatazero + number_of_nonzero_bytes x G_txdatanonzero`

# non-legacy-typed transaction
PayloadGas = number_of_bytes * TxDataGas
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

KeyValidationGas is defined as below based on key type,

| Key Type | Gas |
| :--- | :--- |
| Nil | N/A |
| Legacy | 0 |
| Fail | 0 |
| Public | 0 |
| MultiSig | \(keys-1\) \* GasValidationPerKey \(15000\) |
| RoleBased | Based on keys in the role used in the validation |

KeyCreationGas is defined as below based on key type,

| Key Type | Gas |
| :--- | :--- |
| Nil | N/A |
| Legacy | 0 |
| Fail | 0 |
| Public | GasCreationPerKey \(20000\) |
| MultiSig | \(keys\) \* GasCreationPerKey |
| RoleBased | Gas fee calculated based on keys in each role. e.g., GasRoleTransaction = \(keys\) _GasCreationPerKey_ _GasRoleAccountUpdate = \(keys\)_ GasCreationPerKey GasRoleFeePayer = \(keys\) \* GasCreationPerKey |
