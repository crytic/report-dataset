## Insufficient event generation

**Description**

Events generated during contract execution aid in monitoring, baselining of behavior, and detection of suspicious activity. Without events, users and blockchain-monitoring systems cannot easily detect behavior that falls outside the baseline conditions. This may lead to missed discovery of malfunctioning contracts or malicious attacks.

Multiple critical operations do not emit events. As a result, it will be difficult to review the correct behavior of the contracts once they have been deployed.

The following operations should trigger events:
- Constructor should emit DefaultTimelockParametersSet event
- setPriceOracle
- setManualOverridePrice


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

- SpoolAccessControl.grantSmartVaultOwnership
- ActionManager.setActions
- SmartVaultManager.registerSmartVault
- SmartVaultManager.removeStrategy

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

 - The setAllowedZone function in seaport/contracts/ImmutableSeaport.sol

**Recommendations**

Short term, add events for all functions that change state to aid in better monitoring and alerting.

Long term, ensure that all state-changing operations are always accompanied by events. In addition, use static analysis tools such as Slither to help prevent such issues in the future.


