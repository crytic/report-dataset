## Risks associated with oracle outages

  

**Description**

Under extreme market conditions, the Chainlink oracle may cease to work as expected, causing unexpected behavior in the Increment Protocol.

  

Such oracle issues have occurred in the past. For example, during the LUNA market crash, the Venus protocol was exploited because Chainlink stopped providing up-to-date prices. The interruption occurred because the price of LUNA dropped below the minimum price (minAnswer) allowed by the LUNA / USD price feed on the BNB chain. As a result, all oracle updates reverted. Chainlink’s automatic circuit breakers, which pause price feeds during extreme market conditions, could pose similar problems.

  

Note that these kinds of events cannot be tracked on-chain. If a price feed is paused, updatedAt will still be greater than zero, and answeredInRound will still be equal to roundID.

  

Thus, the Increment Finance team should implement an off-chain monitoring solution to detect any anomalous behavior exhibited by Chainlink oracles. The monitoring solution should check for the following conditions and issue alerts if they occur, as they may be indicative of abnormal market events:

  ● An asset price that is approaching the minAnswer or maxAnswer value
   
   ● The suspension of a price feed by an automatic circuit breaker
   
   ● Any large deviations in the price of an asset

## Risk of oracle outages

**Description**

Under extreme market conditions, the Chainlink oracle may cease to work as expected, causing unexpected behavior in the Fraxlend protocol.

Such oracle issues have occurred in the past. For example, during the LUNA market crash, the Venus protocol was exploited because Chainlink stopped providing up-to-date prices. The interruption occurred because the price of LUNA dropped below the minimum price (minAnswer) allowed by the LUNA/USD price feed on the BNB chain. As a result, all oracle updates reverted. Chainlink’s automatic circuit breakers, which will pause price feeds during extreme market conditions, could pose similar problems.

Note that these kinds of events cannot be tracked on-chain. If a price feed is paused, updatedAt will still be greater than zero, and answeredInRound will still be equal to roundId.

Therefore, the Frax Finance team should implement an off-chain monitoring solution to detect any anomalous behavior exhibited by Chainlink oracles.

**Recommendations**

Short term, implement an off-chain monitoring solution that checks for the following conditions and issues alerts if they occur, as they may be indicative of abnormal market events:
● An asset price that is approaching the minAnswer or maxAnswer value
● The suspension of a price feed by an automatic circuit breaker
● Any large deviations in the price of an asset

## TimelockControllerEmergency could be used to allow anyone to execute proposals

**Description**

The timelock contract can be configured to use open roles, which allow any user to run proposals.

The TimelockControllerEmergency contract is implemented using OpenZeppelin’s TimelockController contract to add a delay for governance decisions to be executed

The OpenZeppelin code implements two functions to execute proposals using "open roles":

The use of open roles means that if the zero address is added with the executor role, then anyone will be able to execute proposals. If open roles is a feature that should never be used, then it should be disallowed when the TimelockControllerEmergency contract is deployed.

However, the contract’s constructor does not contain code to prevent the use of the zero address and, therefore, open roles

**Exploit Scenario**

Alice creates a TimelockControllerEmergency contract, which is misconfigured using one zero address. As a result, Eve is allowed to run proposals.

**Recommendations**

Short term, either properly document the open roles feature or disallow the use of zero addresses when setting roles.

Long term, carefully review and document the assumptions and limitations regarding third-party code integrations and consider whether the limitations are acceptable and whether the assumptions hold.

## Contract architecture is overcomplicated

**Description**

The codebase relies on a significant amount of cross-contract validation, obscure storage solutions, and a series of confusing branches, which obscures the understandability of the codebase.

For example, the getPositionCurrentValue function in the TickLogic contract determines the value, depending on the state of the position. As a result, the number of branches becomes difficult to follow and reason about. This pattern is repeated across the codebase, which is especially problematic with conversions to different precision types.

**Recommendations**

Short term, wherever possible, simplify functions to be as small and self-contained as possible and clearly document all expectations in the codebase.

Long term, thoroughly document the expected flow of each contract, the contracts’ expected interactions with each other, and function call stacks to ensure that all contracts are easy to understand.

## Minting funds to the Hyper contract arbitrarily increases the next caller’s balance

**Description**

When a user mints funds to the Hyper contract, the contract relies on a calculation of the difference between its physical balance and its virtual balance of the given token. However, this calculation increases the Hyper contract’s reserves and the next caller’s balance.

To add tokens into the system, users call the fund function; this function uses the _settlement() function, which calls the settle() function. This function uses the getNetBalance() function, which calculates the difference between the return value of the token.balanceOf function and the Hyper contract’s reserves of the token.

**Exploit Scenario**

Eve, an attacker, creates a DRP token. With her minting rights, she mints 1 million DRP to the Hyper contract. As a result, when Alice funds her account, the Hyper contract’s reserve balance and Alice’s tracked token balance increase by 1 million DRP, along with the tokens she intended to fund.

**Recommendations**

Short term, document the fact that the Hyper reserves and the respective user’s balance for the airdropped token will be attributed to the next user.

Long term, clearly identify the expected and unexpected flows in the contract to ensure that users are aware of expected behavior.

## Incorrect constant for 1000-year periods

**Description**

The Raft finance contracts rely on computing the exponential decay to determine the correct base rate for redemptions. In the MathUtils library, a period of 1000 years is chosen as the maximum time period for the decay exponent to prevent an overflow. However, the _MINUTES_IN_1000_YEARS constant used is currently incorrect:

**Recommendations**

Short term, change the code to compute the _MINUTES_IN_1000_YEARS constant as 1000 * 365 days / 1 minutes .

Long term, improve unit test coverage to uncover edge cases and ensure intended behavior throughout the system. Integrate Echidna and smart contract fuzzing in the system to triangulate subtle arithmetic issues.

## Incorrect constant value for MAX_REDEMPTION_SPREAD

**Description**

The Raft protocol allows a user to redeem their R tokens for underlying wstETH at any time. By doing so, the protocol ensures that it maintains overcollateralization. The redemption spread is part of the redemption rate, which changes based on the price of the R token to incentivize or disincentivize redemption. However, the documentation says that the maximum redemption spread should be 100% and that the protocol will initially set it to 100%.

In the code, the MAX_REDEMPTION_SPREAD constant is set to 2%, and the redemptionSpread variable is set to 1% at construction. This is problematic because setting the rate to 100% is necessary to effectively disable redemptions at launch.

**Exploit Scenario**

The protocol sets the redemption spread to 2%. Alice, a borrower, redeems her R tokens for some underlying wstETH, despite the developers’ intentions. As a result, the stablecoin experiences significant volatility.

**Recommendations**

Short term, set the MAX_REDEMPTION_SPREAD value to 100% and set the redemptionSpread variable to MAX_REDEMPTION_SPREAD in the PositionManager contract’s constructor.

Long term, improve unit test coverage to identify incorrect behavior and edge cases in the protocol.

## Spamming risk in propose functions

**Description**

Anyone with enough veFXS tokens to meet the proposal threshold can submit an unbounded number of proposals to both the FraxGovernorAlpha and FraxGovernorOmega contracts.

The only requirement for submitting proposals is that the msg.sender address must have a balance of veFXS tokens larger than the _proposalThreshold value. Once that requirement is met, a user can submit as many proposals as they would like. A large volume of proposals may create difficulties for off-chain monitoring solutions and user-interface interactions.

**Exploit Scenario**

Mallory has 100,000 voting power. She submits one million proposals with small but unique changes to the description field of each one. The system saves one million unique proposals and emits one million ProposalCreated events. Front-end components and off-chain monitoring systems are spammed with large quantities of data.

**Recommendations**

Short term, track and limit the number of proposals a user can have active at any given time.

Long term, consider cases of user interactions beyond just the intended use cases for potential malicious behavior.


## liquidatableCollateralRatio update can force liquidation without warning

**Description**

The DEFAULT_ADMIN_ROLE can update the liquidatableCollateralRatio without notifying users who hold positions in the rebalance pool. This action could potentially trigger or hinder liquidations without giving users a chance to make adjustments.

Only the DEFAULT_ADMIN_ROLE , which is a multisig wallet address controlled by the AladdinDAO development team, can call the update function for this important system parameter. This implies that updates will take effect immediately, without prior notice to protocol users.

The liquidatableCollateralRatio is used in the liquidate() function to benchmark the system's health and determine if a liquidation is required.

the liquidatableCollateralRatio were raised while the _treasury.collateralRatio() stays the same, a system in good standing could suddenly change so that the liquidate function can be successfully run.

**Exploit Scenario**

Bob, an f(x) protocol user, has a leveraged position of xETH that is close to the liquidation threshold. The AladdinDAO team calls updateLiquidatableCollateralRatio() and raises the liquidatableCollateralRatio variable. The LIQUIDATOR_ROLE calls liquidate() and Bob, along with other users’ positions, lose value.

**Recommendations**

Short term, give access control to a role that is managed by a governance contract where system parameter changes are subject to a timelock. Enforce a minimum and maximum bound for the liquidatableCollateralRatio to provide users with a clear expectation of any potential updates.

Long term, when important protocol parameters are updated, it is crucial to allow users to adjust their positions or exit the system before these updates take effect.

## Time-weighted Chainlink oracle can report inaccurate price

**Description**

The Chainlink oracle used to price the value of ETH in USD is subject to a time-weighted average price that will over-represent or under-represent the true price in times of swift volatility. This can increase the likelihood of bad debt if a sudden price change is not acted on due to the lag, and it is too late once the collateral’s loss in value is reflected in the TWAP price.

**Exploit Scenario**

The price of Ethereum jumps 10% over the course of 10 minutes. Eve notices the price discrepancy between the actual price and the reported price, and takes out a 4x long position on the frxETH pool. Once the time-weighted average price of ETH catches up to the actual price, Eve cashes out her long position for a profit.

**Recommendations**

Short term, remove the reliance on a time-weighted average price, at least for liquidations. It is best to be punitive when pricing debt and conservative when valuing collateral, so the TWAP may be appropriate for minting.

Long term, adhere to best practices for oracle solutions and ensure backup oracles and safety checks do not create credit risk.


