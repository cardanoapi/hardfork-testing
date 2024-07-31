# Migrating Cardano-Marketplace from Plutus V2 to Plutus V3

# 1. DApp Overview

[Cardano Marketplace](https://github.com/dQuadrant/cardano-marketplace) is a straightforward DApp for an NFT marketplace utilizing Plutus smart contracts. It supports minting assets, placing them on sell in the marketplace, buying them from the marketplace, and withdrawing them if you are the asset owner. Additionally, it can perform these tasks with reference scripts.

## 1.1 DApp Types

- Simple Marketplace

    - Basic implementation for buying, selling, and withdrawing assets.

- Configurable Marketplace

    - Adds functionality for an additional fee payable to the market operator, with configurable fee amount and operator address.

The Cardano marketplace has been implemented using Plutus V1, V2, and V3. This article serves as a migration guide from Plutus V2 to Plutus V3.

# 2. Effects of Chang Hard Fork on DApps and Smart Contracts

### 2.1. CIP-0069
A significant change with the new Plutus dependency, as introduced by the [CIP-0069](https://developers.cardano.org/docs/governance/cardano-improvement-proposals/cip-0069/), affects the arguments given to all types of Plutus scripts:


> "This CIP unifies the arguments given to all types of Plutus scripts currently available (spending, certifying, rewarding, minting) by removing the argument of a datum."

Developers must update their DApps if they wish to use the latest dependencies. Previously, all script purposes followed the form `Redeemer -> ScriptContext -> a`, except spending scripts which were `Datum -> Redeemer -> ScriptContext -> a`. With the changes from CIP-0069, both Redeemer and Datum can be decoded from ScriptContext, making the validator take only one argument: `ScriptContext -> a`. This same update applies to Cardano-Marketplace.

Additionally, the requirement for a datum in spending scripts has been made optional. This allows donations and contributions to DAOs to be made more efficiently, as script UTxOs no longer need a datum to be redeemed. DApps can also introduce features that enable users to claim or withdraw funds that were sent without a datum.

### 2.2 References

- **[Plutus V2 ScriptContext](https://github.com/IntersectMBO/plutus/blob/8c7a5f6aa80fff036537d4b130052ea9afe6c28b/plutus-ledger-api/src/PlutusLedgerApi/V2/Contexts.hs#L107)**

- **[Plutus V3 ScriptContext](https://github.com/IntersectMBO/plutus/blob/8c7a5f6aa80fff036537d4b130052ea9afe6c28b/plutus-ledger-api/src/PlutusLedgerApi/V3/Contexts.hs#L438)**

- **[Plutus V3 ScriptInfo](https://github.com/IntersectMBO/plutus/blob/8c7a5f6aa80fff036537d4b130052ea9afe6c28b/plutus-ledger-api/src/PlutusLedgerApi/V3/Contexts.hs#L360)**

In V3, the scriptInfo can identify whether the script is for minting, spending, rewarding, certifying, voting, or proposing. Details about the datum can be obtained if the script is a spending script. This update changes the 3-parameter validator to a 1-parameter validator. The validator still takes an extra argument if the smart contract is to be parameterized. Also, the check function now returns a `BuiltinUnit` data type, which is necessary for a V3 script. Else,a  `PlutusScriptError` stating `“Script returned invalid type”` will be returned while trying to use the script. Thus, a validator for a smart contract with V3 dependency looks like this:

```hs
validator :: BuiltinData -> BuiltinUnit
validator context = check $ validatorLogic (unsafeFromBuiltinData context) 
    where
        vaildatorLogic :: ScriptContext -> Bool
        vaildatorLogic = ... --decoding of the script-context for datum and redeemer information can be done here
```
For smart contracts using Plutus V2, the validator still takes 3 arguments. For example:

```hs
validator :: BuiltinData -> BuiltinData -> BuiltinData -> BuiltinUnit
validator datum redeemer context = 
    check $ 
        validatorLogic 
            (unsafeFromBuiltinData datum)
            (unsafeFromBuiltinData redeemer)
            (unsafeFromBuiltinData context)
    where 
        validatorLogic :: Datum -> Redeemer -> ScriptContext -> Bool 
        validatorLogic datum redeemer context = ...
```
> ***Note***: *The return type for Plutus V2 validators can be anything, and V2 scripts will succeed as long as the evaluation completes without error and does not exceed the budget. This is also true for V1 scripts.*
### 2.3 Reference script cost analysis
DApps using Plutus V1 and V2 scripts need to be aware about the updated transaction costs linked to reference scripts under the Conway era. Also, fees for V2 and V3 will be adjusted following the Chang Hard Fork. This adjustment is due to the requirement to pay for reference scripts and a revision in the cost model.  
The modifications to the cost model are expected to lower fees by approximately 10%-20%, though the introduction of reference script fees will cause a partial increase in overall costs.  
The cost model in Cardano is an integral part of its security protocol. By imposing higher costs on larger reference scripts, the network effectively mitigates spam.
Please refer to the [reference script cost calculator](https://docs.google.com/spreadsheets/d/1KFJCCbkDE5GaghlD4rDXB12pqLKnDFUNOKi0WErp_-Q/edit?gid=0#gid=0) to analyze and compare the cost implications of using reference scripts in Cardano transactions between the Babbage and Conway eras. It could also be a resource for helping developers optimize their DApps for cost efficiency and resource utilization.

# 3. What To Do

If developers want to use `PlutusLedgerApi.V2` dependencies, one of the changes needed is to the return type of the validator if the `check` function is used. Since the type signature for the `check` function has changed from `Bool -> ()` to `Bool -> BuiltinUnit`, the validator implementing this function must also return `BuiltinUnit`. This type is compatible with the `PlutusTx.compile` function. 

However, this is not mandatory for V2 scripts, as there are ways to bypass the `check` function and return void directly. The following technique uses if-then-else statement to avoid using the check function:

```hs
validator :: BuiltinData -> BuiltinData -> BuiltinData -> ()
validator datum redeemer context = 
    if
        validatorLogic 
            (unsafeFromBuiltinData datum)
            (unsafeFromBuiltinData redeemer)
            (unsafeFromBuiltinData context)
    then ()
    else traceError "validator: Validation Failed"
    where 
        validatorLogic :: Datum -> Redeemer -> ScriptContext -> Bool 
        validatorLogic datum redeemer context = ...
```
The implementation of the `check` function has been tested and verified in the Cardano-Marketplace V2. Here is a sample of the validator using this upgrade:

```hs
{-# INLINABLE mkWrappedMarket #-}
mkWrappedMarket ::  BuiltinData -> BuiltinData -> BuiltinData -> BuiltinUnit
mkWrappedMarket  d r c = 
    check $ 
        mkMarket 
        (parseData d "Invalid data") 
        (parseData r "Invalid redeemer") 
        (parseData c "Invalid context")
  where
    parseData md s = case fromBuiltinData  md of 
      Just datum -> datum
      _      -> traceError s
```
To see the full implementation, visit [Cardano-Marketplace-V2/SimpleMarketplace](https://github.com/dQuadrant/cardano-marketplace/blob/3f0e1290ea79f191511875221f41c9a90d852361/marketplace-plutus/Plutus/Contracts/V2/SimpleMarketplace.hs#L86)

Another adjustment developers need to implement is adding the following OPTIONS_GHC pragma, specifying the target version 1.0.0, to their V2 script files:

```hs
{-# OPTIONS_GHC -fplugin-opt PlutusTx.Plugin:target-version=1.0.0 #-}
```
This is necessary because the default target version is set to 1.1.0, and V1 and V2 scripts are not compatible with this version. An example of the addition of this pragma for V2 plutus script is available in [Cardano-Marketplace-V2/SimpleMarketplace](https://github.com/dQuadrant/cardano-marketplace/blob/80704834db82ed028d855777a49bbab7cbde9071/marketplace-plutus/Plutus/Contracts/V2/SimpleMarketplace.hs#L20). 

All parameterized scripts require the parameter to be compiled at runtime to Plutus Core. DApps that use parameterized V2 contracts often use the `unsafeApplyCode` and `applyCode` function and must also use `liftCode plcVersion100` instead of `liftCodeDef` as the default is the latest version which is `plcVersion110`, which is again, incompatible with V1 and V2 scripts. A parameterized V2 contract is implemented in the configurable marketplace and the validator is the following: 

```hs
configurableMarketValidator constructor = 
  $$(PlutusTx.compile [|| mkWrappedConfigurableMarket ||])
    `unsafeApplyCode` PlutusTx.liftCode plcVersion100 constructor
```
A working implementation is available at [Cardano-Marketplace/V2/ConfigurableMarketplace](https://github.com/dQuadrant/cardano-marketplace/blob/80704834db82ed028d855777a49bbab7cbde9071/marketplace-plutus/Plutus/Contracts/V2/ConfigurableMarketplace.hs#L206)

For developers intending to use `PlutusLedgerApi.V3`, there needs to be additional changes in the smart contract code. A similar implementation using PlutusV3 for a simple marketplace can be found here: [Cardano-Marketplace-V3/SimpleMarketplace](https://github.com/dQuadrant/cardano-marketplace/blob/3f0e1290ea79f191511875221f41c9a90d852361/marketplace-plutus/Plutus/Contracts/V3/SimpleMarketplace.hs#L122). In this implementation, the validator looks like this:

```hs
{-# INLINABLE mkWrappedMarket #-}
mkWrappedMarket ::  BuiltinData -> BuiltinUnit
mkWrappedMarket ctx = check $ mkMarket (parseData ctx "Invalid context")
```
It takes only one argument, `BuiltinData`, which is parsed into the `ScriptContext`.

The update has been catered towards V3 plutus scripts, as there is no need to add additional pragma on plutus smarts contracts using `PlutusLedgerApi.V3` dependencies, and also, there is no need to specify plc versions while applying parameters. However, if the developer prefers, the following pragma can be added on V3 smart contracts: 

```hs
{-# OPTIONS_GHC -fplugin-opt PlutusTx.Plugin:target-version=1.1.0 #-}
```
In the case of parameterized V3 contracts, `PlutusTx.iftCodeDef` can directly be used, but again, if desired, the validator can be written in the following way by explicitly specifying `plcVersion110`:

```hs
configurableMarketValidator constructor = 
  $$(PlutusTx.compile [|| mkWrappedConfigurableMarket ||])
    `unsafeApplyCode` PlutusTx.liftCode plcVersion110 constructor
```
The [Cardano-Marketplace/V3/ConfigurableMarketplace](https://github.com/dQuadrant/cardano-marketplace/blob/80704834db82ed028d855777a49bbab7cbde9071/marketplace-plutus/Plutus/Contracts/V3/ConfigurableMarketplace.hs) uses a parameterized V3 contract with the optional pragma and specifies the plc version in the validator.

## 3.1 Decoding Datum and Redeemer

Decoding the datum and the redeemer now depends on the script's purpose. Mainly, DApps are developed with scripts whose purpose is to either mint, spend, certify, or reward. V3 offers additional purposes, including voting and proposing, related to governance actions introduced in the Chang hard fork. Let's look into decoding datum and redeemer.

### 3.1.1 Decoding Datum

Datum is only needed when the script is of the spending type. Therefore, the datum will come from the `scriptInfo` in the `scriptContext`. Here is a simple example where we expect the script to be of the spending type and get the datum:

```hs
ctx :: ScriptContext -- PlutusV3 script-context

scriptInfo :: ScriptInfo
scriptInfo = scriptContextScriptInfo ctx

datum :: Datum
datum = case scriptInfo of 
    SpendingScript outRef datum -> case datum of
          Just d ->  case fromBuiltinData (getDatum d) of
            Nothing -> traceError "Invalid datum format"
            Just v -> v
          Nothing -> traceError "Missing datum"
    _ -> traceError "Script Purpose Mismatch"
```
### 3.1.2 Decoding Redeemer

For decoding the redeemer, we don't need to expect the script purpose to be of a particular type. It can be directly decoded from the scriptContext. For a scriptContext ctx, the redeemer can be extracted in the following way:

```hs
redeemer :: Redeemer
redeemer = case fromBuiltinData $ getRedeemer (scriptContextRedeemer ctx) of
    Nothing -> traceError "Invalid Redeemer"
    Just r -> r
```
Just like parsing datum to a correct data type, we can parse the redeemer to `BuiltinData` using `getRedeemer` first and then to the required data type using `fromBuiltinData` or `unsafeFromBuiltinData`.

For the simple cardano marketplace contract using PlutusV3, the validator logic function is available at [Cardano-Marketplace-V3/SimpleMarketplace](https://github.com/dQuadrant/cardano-marketplace/blob/3f0e1290ea79f191511875221f41c9a90d852361/marketplace-plutus/Plutus/Contracts/V3/SimpleMarketplace.hs#L80). This function takes ScriptContext only, with the decoding of datum and redeemer done inside the function.

# 4. Changes In Efficiency

With more functionalities introduced in PlutusV3 for governance actions, the `ScriptContext` has additional fields. Because of this, the size of the script (in bytes) of smart contracts written using `PlutusLedgerApi.V3` has increased a considerable amount.

A workaround to this problem is to write the validator lazily, not decoding the script-context entirely, but by only parsing the required fields contained within the script-context.

The script-context initially will be of the form `BuiltinData`. We can then change this to a BuiltinList using the following function:

```hs
import qualified PlutusTx.Builtins.Internal as BI

constrArgs :: BuiltinData -> BI.BuiltinList BuiltinData
constrArgs bd = BI.snd (BI.unsafeDataAsConstr bd)
```
This allows us to extract only the necessary information from the script-context without needing to decode it fully, as the `unsafeFromBuiltinData` function does. But, this comes at the expense of a cleaner code. Indexing of a `BuiltinList` type is not supported in Plutus yet. So, we have to utilize the `BI.head` and `BI.tail` function repeatedly in order to get the element we want. For example, to get the datum and redeemer from script-context, the following approach can be followed:

```hs
ctx :: BuiltinData 

context :: BI.BuiltinList BuiltinData
context = constrArgs ctx

redeemerFollowedByScriptInfo :: BI.BuiltinList BuiltinData
redeemerFollowedByScriptInfo = BI.tail context

redeemerBuiltinData :: BuiltinData
redeemerBuiltinData = BI.head redeemerFollowedByScriptInfo

scriptInfoData :: BuiltinData
scriptInfoData = BI.head (BI.tail redeemerFollowedByScriptInfo)

datumData :: BuiltinData
datumData = BI.head (constrArgs (BI.head (BI.tail (constrArgs scriptInfoData))))

redeemer :: Redeemer
redeemer = unsafeFromBuiltinData redeemerBuiltinData 

datum :: Datum
datum = unsafeFromBuiltinData (getDatum (unsafeFromBuiltinData datumData)) 

```
The lazy validator for the simple marketplace can be found here: [Cardano-Marketplace-V3/SimpleMarketplace-Lazy](https://github.com/dQuadrant/cardano-marketplace/blob/3f0e1290ea79f191511875221f41c9a90d852361/marketplace-plutus/Plutus/Contracts/V3/SimpleMarketplace.hs#L126)

This method can be taken to the next level with super lazy evaluation, where we decode the `txInfo` to obtain the required inputs, outputs, signatures, etc., for the validation logic, while ignoring the unused fields. For instance, the following method can be used to get the inputs, reference inputs, outputs and signature of a transaction from a script-context.

```hs
context = constrArgs ctx

txInfoData :: BuiltinData 
txInfoData = BI.head context

lazyTxInfo :: BI.BuiltinList BuiltinData
lazyTxInfo = constrArgs txInfoData 

inputs :: [TxInInfo]
inputs = parseData (BI.head lazyTxInfo) "txInfoInputs: Invalid [TxInInfo] type"

refInputs :: [TxInInfo]
refInputs = parseData (BI.head (BI.tail lazyTxInfo)) "txInfoReferenceInputs: Invalid [TxInInfo] type"

outputs :: [TxOut] 
outputs = parseData (BI.head (BI.tail (BI.tail lazyTxInfo))) "txInfoOutputs: Invalid [TxOut] type"

signatures :: [PubKeyHash]
signatures = parseData 
    (BI.head  BI.tail $ BI.tail $ BI.tail $ BI.tail $ BI.tail $ BI.tail $ BI.tail  BI.tail lazyTxInfo) 
    "txInfoSignatories: Invalid [PubKeyHash] type"
```
This has a very pronounced effect on the size of a cbor script.

The latest run of the Cardano Marketplace on sanchonet generated a report indicating the following:

## 4.1 Simple Market

| Implementation | ScriptBytes |
|---|---|
| Plutus V2 | 4594 |
| Plutus V2 Super Lazy | 2848 |
| Plutus V3 | 8118 |
| Plutus V3 Lazy | 7125 |
| Plutus V3 Super Lazy | 2573 |

## 4.2 Configurable Market

| Implementation | ScriptBytes |
|---|---|
| Plutus V2 | 3615 |
| Plutus V2 Super Lazy | 2875 |
| Plutus V3 | 7883 |
| Plutus V3 Lazy | 7263 |
| Plutus V3 Super Lazy | 2586 |


> Click [here](https://github.com/dQuadrant/cardano-marketplace/blob/dev/reports/test-sanchonet-cardano-api-9.0.0/2024-07-23_10-52-57-transaction-report.md) to view the full report

# 5. Conclusion

In the migration of Cardano-Marketplace from Plutus V2 to Plutus V3, the adoption of the V3 Super Lazy approach has demonstrated notable efficiency gains. This methodology consistently emerges as the optimal choice across various implementations, including both Simple Market and Configurable Market. The key benefits of the V3 Super Lazy approach include reduced script bytes, lower memory consumption, decreased CPU fees, and minimized transaction bytes.

The shift to Plutus V3 introduces a streamlined validation process, where scripts take a single argument (ScriptContext), simplifying the overall structure. Despite the increase in script size due to additional functionalities for governance actions, the lazy evaluation technique offsets this by decoding only necessary fields, thus enhancing efficiency without fully decoding the script context.

Overall, migrating to Plutus V3, particularly utilizing the Super Lazy evaluation, offers significant improvements in performance and resource utilization, ensuring a more efficient and scalable implementation for the Cardano-Marketplace. This transition not only aligns with the latest advancements in the Cardano ecosystem but also prepares the platform for future enhancements and expanded use cases.
