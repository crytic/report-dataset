## Insufficient event generation

**Description**

Events generated during contract execution aid in monitoring, baselining of behavior, and detection of suspicious activity. Without events, users and blockchain-monitoring systems cannot easily detect behavior that falls outside the baseline conditions. This may lead to missed discovery of malfunctioning contracts or malicious attacks.

Multiple critical operations do not emit events. As a result, it will be difficult to review the correct behavior of the contracts once they have been deployed.

The following operations should trigger events:
● Constructor should emit DefaultTimelockParametersSet event
● setPriceOracle
● setManualOverridePrice


**Exploit Scenario**

The Maple team must use the setManualOverridePrice to manually set the price of ETH due to an oracle outage. However, the users or a monitoring system cannot easily follow the applied changes.

**Recommendations**

Short term, add events for all operations that may contribute to a higher level of monitoring and alerting.

Long term, consider using a blockchain-monitoring system to track any suspicious behavior in the contracts.

## Incorrect GovernorshipAccepted event argument

**Description**

The MapleGlobals contract emits the GovernorshipAccepted event with an incorrect previous owner value.

MapleGlobals implements a two-step process for ownership transfer in which the current owner has to set the new governor, and then the new governor has to accept it. The acceptGovernor function first sets the new governor with _setAddress and then emits the GovernorshipAccepted event with the first argument defined as the old governor and the second the new one. However, because the admin() function returns the current value of the governor, both arguments will be the new governor.

MapleGlobals implements a two-step process for ownership transfer in which the current owner has to set the new governor, and then the new governor has to accept it. The acceptGovernor function first sets the new governor with _setAddress and then emits the GovernorshipAccepted event with the first argument defined as the old governor and the second the new one. However, because the admin() function returns the current value of the governor, both arguments will be the new governor.

**Exploit Scenario**

The Maple team decides to transfer the MapleGlobals’ governor to a new multi-signature wallet. The team has a script that verifies the correct execution by checking the events emitted; however, this script creates a false alert because the GovernorshipAccepted event does not have the expected arguments.

**Recommendations**

Short term, emit the GovernorshipAccepted event before calling _setAddress

Long term, add tests to check the events have the expected arguments.

## Events could be improved

**Description**

The events declared in the Fraxferry contract could be improved to be more useful to users and monitoring systems.

Certain events could be more useful if they used the indexed keyword. For example, in the Embark event, the indexed keyword could be applied to the sender parameter.

Additionally, SetCaptain, SetFirstOfficier, SetFee, and SetMinWaitPeriods could be more useful if they emitted the previous value in addition to the newly set one.

**Recommendations**

Short term, add the indexed keyword to any events that could benefit from it; modify events that report on setter operations so that they report the previous values in addition to the newly set values.

## Lack of events emitted for state-changing functions

**Description**

Finding ID: TOB-SPL-32 Multiple critical operations do not emit events. As a result, it will be difficult to review the correct behavior of the contracts once they have been deployed.

Events generated during contract execution aid in monitoring, baselining of behavior, and detection of suspicious activity. Without events, users and blockchain-monitoring systems cannot easily detect behavior that falls outside the baseline conditions. This may prevent malfunctioning contracts or attacks from being detected.

The following operations should trigger events:

● SpoolAccessControl.grantSmartVaultOwnership
● ActionManager.setActions
● SmartVaultManager.registerSmartVault
● SmartVaultManager.removeStrategy
● SmartVaultManager.syncSmartVault
● SmartVaultManager.reallocate
● StrategyRegistry.registerStrategy
● StrategyRegistry.removeStrategy
● StrategyRegistry.doHardWork
● StrategyRegistry.setEcosystemFee
● StrategyRegistry.setEcosystemFeeReceiver
● StrategyRegistry.setTreasuryFee
● StrategyRegistry.setTreasuryFeeReceiver
● Strategy.doHardWork
● RewardManager.addToken
● RewardManager.extendRewardEmission

**Exploit Scenario**

The Spool system experiences a security incident, but the Spool team has trouble reconstructing the sequence of events causing the incident because of missing log information.

**Recommendations**

Short term, add events for all operations that may contribute to a higher level of monitoring and alerting.

Long term, consider using a blockchain-monitoring system to track any suspicious behavior in the contracts. The system relies on several contracts to behave as expected. A monitoring mechanism for critical events would quickly detect any compromised system components.

## Lack of event generation

**Description**

Multiple critical operations do not emit events. This creates uncertainty among users interacting with the system.

The setRateControlThresholds function in the RootERC20PredicateFlowRate contract does not emit an event when it updates the largeTransferThresholds critical storage variable for a token (figure 4.1). However, having an event emitted to reflect such a change in the critical storage variable may allow other system and off-chain components to detect suspicious behavior in the system.

Events generated during contract execution aid in monitoring, baselining behavior, and detecting suspicious activity. Without events, users and blockchain-monitoring systems cannot easily detect behavior that falls outside the baseline conditions, and malfunctioning contracts and attacks could go undetected.

In addition to the above function, the following function should also emit events:

 - ● The setAllowedZone function in

seaport/contracts/ImmutableSeaport.sol

**Recommendations**

Short term, add events for all functions that change state to aid in better monitoring and alerting.

Long term, ensure that all state-changing operations are always accompanied by events. In addition, use static analysis tools such as Slither to help prevent such issues in the future.

## Lack of event generation

**Description**

Multiple user operations do not emit events. As a result, it will be difficult to review the contracts’ behavior for correctness once they have been deployed.

Events generated during contract execution aid in monitoring, baselining of behavior, and detection of suspicious activity. Without events, users and blockchain-monitoring systems cannot easily detect behavior that falls outside the baseline conditions; malfunctioning contracts and attacks could go undetected.

The following operations should trigger events:

 - ● Splitter ○ Splitter__init
  ○ updatePool
  ○ updateGroupShares 
  ○ removeUpgradeability
 
 - ● L1Sender
  ○ L1Sender__init
  ○ setRewardTokenConfig 
  ○ setDepositTokenConfig 
  ○ updateAllowedAddresses 
  ○ sendDepositToken 
  ○ sendMintMessage
 - ● Distribution
  ○ removeUpgradeability
 - ● L2MessageReceiver 
  ○ setParams
 - ● L2TokenReceiver 
  ○ editParams 
  ○ withdrawToken 
  ○ withdrawTokenId
 - ● RewardClaimer 
  ○ RewardClaimer__init 
  ○ setSplitter 
  ○ setL1Sender
 - ● Token 
  ○ updateMinter

**Exploit Scenario**

An attacker discovers a vulnerability in any of these contracts and modifies its execution. Because these actions generate no events, the behavior goes unnoticed until there is follow-on damage, such as financial loss.

**Recommendations**

Short term, add events for all operations that could contribute to a higher level of monitoring and alerting.

Long term, consider using a blockchain-monitoring system to track any suspicious behavior in the contracts. The system relies on several contracts to behave as expected. A monitoring mechanism for critical events would quickly detect any compromised system components.

## Missing event emission

**Description**

The critical operation updateOnchainSpotEncodings does not emit an event. Having an event emitted to reflect changes to this critical storage variable will allow other system/off-chain components to detect suspicious behavior in the system.

Events generated during contract execution aid in monitoring, baselining of behavior, and detecting suspicious activity. Without events, users and blockchain-monitoring systems cannot easily detect behavior that falls outside the baseline conditions; malfunctioning contracts and attacks could go undetected.

**Recommendations**

Short term, emit an event in the updateOnchainSpotEncodings function.

Long term, ensure all state-changing operations are always accompanied by events. In addition, use static analysis tools such as Slither to help prevent such issues in the future.

## Lack of events for critical operations

**Description**

Two critical operations do not trigger events. As a result, it will be difficult to review the correct behavior of the contracts once they have been deployed.

TheLiquidityPoolProxycontract’ssetImplementationfunction is called to set the implementation address of the liquidity pool and does not emit an event providing confirmation of that operation to the contract’s caller (figure 7.1).

Calls to theupdateSlicesfunction in theProteuscontract do not trigger events either (figure 7.2). This is problematic because updates to theslicesarray have a significant effect on the configuration of the identity curve (TOB-SHELL-1).

Without events, users and blockchain-monitoring systems cannot easily detect suspicious behavior.

**Exploit Scenario**

Eve, an attacker, is able to take ownership of the LiquidityPoolProxy contract. She then sets a new implementation address. Alice, a Shell Protocol team member, is unaware of the change and does not raise a security incident.

**Recommendations**

Short term, add events for all critical operations that result in state changes. Events aid in contract monitoring and the detection of suspicious behavior.

Long term, consider using a blockchain-monitoring system to track any suspicious behavior in the contracts. The system relies on several contracts to behave as expected. A monitoring mechanism for critical events would quickly detect any compromised system components.



