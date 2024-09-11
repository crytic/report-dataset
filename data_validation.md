## Inconsistent lower bounds on collateral weights

  

**Description**

The lower bound on a collateral asset’s initial weight (when the collateral is first whitelisted) is different from that enforced if the weight is updated; this discrepancy increases the likelihood of collateral seizures by liquidators.

  

A collateral asset’s weight represents the level of risk associated with accepting that asset as collateral. This risk calculation comes into play when the protocol is assessing whether a liquidator can seize a user’s non-UA collateral. To determine the value of each collateral asset, the protocol multiplies the user’s balance of that asset by the collateral weight (a percentage). A riskier asset will have a lower weight and thus a lower value. If the total value of a user’s non-UA collateral is less than the user’s UA debt, a liquidator can seize the collateral.

When whitelisting a collateral asset, the Perpetual.addWhiteListedCollateral function requires the collateral weight to be between 10% and 100% (figure 2.1). According to the documentation, these are the correct bounds for a collateral asset’s weight.

  

However, governance can choose to update that weight via a call to Perpetual.changeCollateralWeight, which allows the weight to be between 1% and 100%

If the weight of a collateral asset were mistakenly set to less than 10%, the value of that collateral would decrease, thereby increasing the likelihood of seizures of non-UA collateral.

  

**Exploit Scenario**

Alice, who holds the governance role, decides to update the weight of a collateral asset in response to volatile market conditions. By mistake, Alice sets the weight of the collateral to 1% instead of 10%. As a result of this change, Bob’s non-UA collateral assets decrease in value and are seized.

  

**Recommendations**

Short term, change the lower bound on newWeight in the changeCollateralWeight function from 1e16 to 1e17.

Long term, expand the unit test suite to cover additional edge cases and to ensure that the system behaves as expected.

  
  
  

## Funding payments are made in the wrong token

  

**Description**

The funding payments owed to users are made in vBase instead of UA tokens; this results in incorrect calculations of users’ profit-and-loss (PnL) values, an increased risk of liquidations, and a delay in the convergence of a Perpetual contract’s value with that of the underlying base asset.

  

When the protocol executes a trade or liquidity provision, one of its first steps is settling the funding payments that are due to the calling user. To do that, it calls the _settleUserFundingPayments function in the ClearingHouse contract (figure 6.1). The function sums the funding payments due to the user (as a trader and / or a liquidity provider) across all perpetual markets. Once the function has determined the final funding payment due to the user (fundingPayments), the Vault contract’s settlePnL function changes the UA balance of the user.

  

Both the Perpetual.settleTrader and Perpetual.settleLp functions internally call _getFundingPayments to calculate the funding payment due to the user for a given market (figure 6.2).

  

However, the upcomingFundingPayment value is expressed in vBase, since it is the product of a percentage, which is unitless, and a vBase token amount, vBaseAmountToSettle. Thus, the fundingPayments value that is calculated in _settleUserFundingPayments is also expressed in vBase. However, the settlePnL function internally updates the user’s balance of UA, not vBase. As a result, the user’s UA balance will be incorrect, since the user’s profit or loss may be significantly higher or lower than it should be. This discrepancy is a function of the price difference between the vBase and UA tokens.

  

The use of vBase tokens for funding payments causes three issues. First, when withdrawing UA tokens, the user may lose or gain much more than expected. Second, since the UA balance affects the user’s collateral reserve total, the balance update may increase or decrease the user’s risk of liquidation. Finally, since funding payments are not made in the notional asset, the convergence between the mark and index prices may be delayed.

  

**Exploit Scenario**

The BTC / USD perpetual market’s mark price is significantly higher than the index price. Alice, who holds a short position, decides to exit the market. However, the protocol calculates her funding payments in BTC and does not convert them to their UA equivalents before updating her balance. Thus, Alice makes much less than expected.

  

**Recommendations**

Short term, use the vBase.indexPrice() function to convert vBase token amounts to UA before the call to vault.settlePnL.

  

Long term, expand the unit test suite to cover additional edge cases and to ensure that the system behaves as expected.

  
  

## Excessive dust collection may lead to premature closures of long positions

  

**Description**

The upper bound on the amount of funds considered dust by the protocol may lead to the premature closure of long positions.

  

The protocol collects dust to encourage complete closures instead of closures that leave a position with a small balance of vBase. One place that dust collection occurs is the Perpetual contract’s _reducePositionOnMarket function (figure 7.1).

  

If netPositionSize, which represents a user’s position after its reduction, is between 0 and 1e17 (1/10 of an 18-decimal token), the system will treat the position as closed and donate the dust to the insurance protocol. This will occur regardless of whether the user intended to reduce, rather than fully close, the position. (Note that netPositionSize is positive if the overall position is long. The dust collection mechanism used for short positions is discussed in TOB-INC-11.)

  

However, if netPositionSize is tracking a high-value token, the donation to Insurance will no longer be insignificant; 1/10 of 1 vBTC, for instance, would be worth ~USD 2,000 (at the time of writing). Thus, the donation of a user’s vBTC dust (and the resultant closure of the vBTC position) could prevent the user from profiting off of a ~USD 2,000 position.

  

**Exploit Scenario**

Alice, who holds a long position in the vBTC / vUSD market, decides to close most of her position. After the swap, netPositionSize is slightly less than 1e17. Since a leftover balance of that amount is considered dust (unbeknownst to Alice), her ~1e17 vBTC tokens are sent to the Insurance contract, and her position is fully closed.

  

**Recommendations**

Short term, have the protocol calculate the notional value of netPositionSize by multiplying it by the return value of the indexPrice function. Then have it compare that notional value to the dust thresholds. Note that the dust thresholds must also be expressed in the notional token and that the comparison should not lead to a significant decrease in a user’s position.

  

Long term, document this system edge case to inform users that a fraction of their long positions may be donated to the Insurance contract after being reduced.

  
  
  
  

## Accuracy of market and oracle TWAPs is tied to the frequency of user interactions

  

**Description**

The oracle and market TWAPs can be updated only during traders’ and liquidity providers’ interactions with the protocol; a downtick in user interactions will result in less accurate TWAPs that are more susceptible to manipulation.

  

The accuracy of a TWAP is related to the number of data points available for the average price calculation. The less often prices are logged, the less robust the TWAP becomes. In the case of the Increment Protocol, a TWAP can be updated with each block that contains a trader or liquidity provider interaction. However, during a market slump (i.e., a time of reduced network traffic), there will be fewer user interactions and thus fewer price updates.

  

TWAP updates are performed by the Perpetual._updateTwap function, which is called by the internal Perpetual._updateGlobalState function. Other protocols, though, take a different approach to keeping markets up to date. The Compound Protocol, for example, has an accrueInterest function that is called upon every user interaction but is also a standalone public function that anyone can call.

  

**Recommendations**

Short term, create a public updateGlobalState function that anyone can call to internally call _updateGlobalState.

  

Long term, create an off-chain worker that can alert the team to periods of perpetual market inactivity, ensuring that the team knows to update the market accordingly.

  
  
  
  

## Liquidations of short positions may fail because of insufficient dust collection

  

**Description**

Because the protocol does not collect the dust associated with short positions, attempts to fully close and then liquidate those positions will fail.

  

One of the key requirements for the successful liquidation of a position is the closure of the entire position; in other words, by the end of the transaction, the debt and assets of the trader or liquidity provider must be zero. The process of closing a long position is a straightforward one, since identifying the correct proposedAmount value (the amount of tokens to be swapped) is trivial. Finding the correct proposedAmount for a short position, however, is more complex.

  

If the proposedAmount estimate is incorrect, the transaction will result in leftover dust, which the protocol will attempt to collect (figure 11.1).

  

The protocol will collect leftover dust only if netPositionSize is greater than zero, which is possible only if the position that is being closed is a long one. If a short position is left with any dust, it will not be collected, since netPositionSize will be less than zero.

  

This inconsistency has a direct impact on the success of liquidations, because a position must be completely closed in order for a liquidation to occur; no dust can be left over. When liquidating the position of a liquidity provider, the Perpetual contract’s _settleLpPosition function checks whether netBasePosition is less than zero (as shown in figure 11.2). If it is, the liquidation will fail. Because the protocol does not collect dust from short positions, the netBasePosition value of such a position may be less than zero. The ClearingHouse._liquidateTrader function, called to liquidate traders’ positions, enforces a similar requirement regarding total closures.

  

If the liquidation of a position fails, any additional attempts at liquidation will lower the liquidator’s profit margin, which might dissuade the liquidator from trying again. Additionally, failed liquidations prolong the protocol’s exposure to debt.

  

**Exploit Scenario**

Alice, a liquidator, notices that a short position is no longer valid and decides to liquidate it. However, Alice sets an incorrect proposedAmount value, so the position is left with some dust. Because the protocol does not collect the dust of short positions, the liquidation fails. As a result, Alice loses money—and the loss dissuades her from attempting to liquidate any other undercollateralized positions.

**Recommendations**

Short term, take the following steps:

1.  Implement the short-term recommendation outlined in TOB-INC-7 to prevent the collection of an excessive amount of dust.
    
2.  When the protocol is liquidating a short position, take the absolute value of netPositionSize and check whether it can be considered dust. If it can, zero out the position’s balance, but do not donate the position’s balance to the Insurance contract. A non-zero netPositionSize for a short position means that the position holds a debt, and that debt should not be transferred to insurance.
    
3.  Remove the checks of netBasePosition from the _settleLpPosition function. (The changes made in the first two steps will render them redundant.)
    
4.  Add a check of the _isTraderPositionOpen function’s return value at the end of the _liquidateLp function to ensure that the account’s openNotional and positionSize values are equal to zero.
    

  

Long term, implement the long-term recommendation outlined in TOB-INC-7. Additionally, document the fact that a liquidator should use the CurveCryptoViews.get_dy_ex_fees function to obtain an accurate estimate of the proposedAmount value before attempting to close a short position.

## The protocol could stop working prematurely

**Description**

The _uint48 function is incorrectly implemented such that it requires the input to be less than or equal to type(uint32).max instead of type(uint48).max. This could lead to the incorrect reversion of successful executions.

The function is mainly used to keep track of when each loan’s payment starts and when it is due. All variables for which the result of _uint48 is assigned are effectively of uint48 type.

**Exploit Scenario**

The protocol stops working when we reach a block.timestamp value of type(uint32).max instead of the expected behavior to work until the block.timestamp reaches a value of type(uint48).max.

**Recommendations** 

Short term, correct the _uint48 implementation by checking that the input_ is less than the type(uint48).max and that it returns an uint48 type.

Long term, improve the unit-tests to account for extreme but valid states that the protocol supports.

## Partially incorrect Chainlink price feed safety checks

**Description**

The getLatestPrice function retrieves a specific asset price from Chainlink. However, the price (a signed integer) is first checked that it is non-zero and then is cast to an unsigned integer with a potentially negative value. An incorrect price would temporarily affect the expected amount of fund assets during liquidation.

**Exploit Scenario**

Chainlink’s oracle returns a negative value for an in-process liquidation. This value is then unsafely cast to an uint256. The expected amount of fund assets from the protocol is incorrect, which prevents liquidation.

**Recommendations**

Short term, check that the price is greater than 0.

Long term, add tests for the Chainlink price feed with various edge cases. Additionally, set up a monitoring system in the event of unexpected market failures. A Chainlink oracle can have a minimum and maximum value, and if the real price is outside of that range, it will not be possible to update the oracle; as a result, it will report an incorrect price, and it will be impossible to know this on-chain.

## Inaccurate accounting of unrealizedLosses during default warning revert

**Description**

During the process of executing the removeDefaultWarning function, an accounting discrepancy fails to decrement netLateInterest from unrealizedLosses, resulting in an over-inflated value.

The triggerDefaultWarning function updates unrealizedLosses with the defaulting loan’s principal_, netInterest_, and netLateInterest_ values.

When the warning is removed by the _revertDefaultWarning function, only the values of the defaulting loan’s principal and interest are decremented from unrealizedLosses. This leaves a discrepancy equal to the amount of netLateInterest_.

**Exploit Scenario**

Alice has missed several interest payments on her loan and is about to default. Bob, the poolManager, calls triggerDefaultWarning on the loan to account for the unrealized loss in the system. Alice makes a payment to bring the loan back into good standing, the claim function is triggered, and _revertDefaultWarning is called to remove the unrealized loss from the system. The net value of Alice’s loan’s late interest value is still accounted for in the value of unrealizedLosses. From then on, when users call Pool.withdraw, they will have to exchange more shares than are due for the same amount of assets.

**Recommendations**

Short term, add the value of netLateInterest to the amount decremented from unrealizedLosses when removing the default warning from the system.

Long term, implement robust unit-tests and fuzz tests to validate math and accounting flows throughout the system to account for any unexpected accounting discrepancies.

## WithdrawalManager can have an invalid exit configuration

**Description**

The setExitConfig function sets the configuration to exit from the pool. However, unsafe casting allows this function to set an invalid configuration.

The function performs a few initial checks; for example, it checks that windowDuration is not 0 and that windowDuration is less than cycleDuration. However, when setting the configuration, the initialCycleId_, initialCycleTime_, cycleDuration_, and windowDuration_ are unsafely casted to uint64 from uint256. In particular, cycleDuration_ and windowDuration_ are user-controlled by the poolDelegate.

**Exploit Scenario**

Bob, the pool delegate, calls setExitConfig with cycleDuration_ equal to type(uint64).max + 1 and windowDuration_ equal to type(uint64).max. The checks pass, but the configuration does not adhere to the invariant windowDuration <= cycleDuration.

**Recommendations**

Short term, safely cast the variables when setting the configuration to avoid any possible errors.

Long term, improve the unit-tests to check that important invariants always hold.

## Loan can be impaired when the protocol is paused

**Description**

The impairLoan function allows the poolDelegate or governor to impair a loan when the protocol is paused due to a missing whenProtocolNotPaused modifier. The role of this function is to mark the loan at risk of default by updating the loan’s nextPaymentDueDate. Although it would be impossible to default the loan in a paused state (because the triggerDefault function correctly has the whenProtocolNotPaused modifier), it is unclear if the other state variable changes would be a problem in a paused system. Additionally, if the protocol is unpaused, it is possible to call removeLoanImpairment and restore the loan’s previous state.

**Exploit Scenario**

Bob, the MapleGlobal security admin, sets the protocol in a paused state due to an unknown occurrence, expecting the protocol’s state to not change and debugging the possible issue. Alice, a pool delegate who does not know that the protocol is paused, calls impairLoan, thereby changing the state and making Bob’s debugging more difficult.

**Recommendations**

Short term, add the missing whenProtocolNotPaused modifier to the impairLoan function.

Long term, improve the unit-tests to check for the correct system behavior when the protocol is paused and unpaused. Additionally, integrate the Slither script in appendix D into the development workflow.

## Fee treasury could go to the zero address

**Description**

The _disburseLiquidationFunds and _distributeClaimedFunds functions, which send the fees to the various actors, do not check that the mapleTreasury address was set.

Althoughthe mapleTreasury address is supposedly set immediately after the creation of the MapleGlobals contract, no checks prevent sending the fees to the zero address, leading to a loss for Maple.

**Exploit Scenario**

Bob, a Maple admin, sets up the protocol but forgets to set the mapleTreasury address. Since there are no warnings, the expected claim or liquidation fees are sent to the zero address until the Maple team notices the issue.

**Recommendations**

Short term, add a check that the mapleTreasury is not set to address zero in _disburseLiquidationFunds and _distributeClaimedFunds.

Long term, improve the unit and integration tests to check that the system behaves correctly both for the happy case and the non-happy case.

## Lack of two-step process for contract ownership changes

**Description**

The owner of a contract that inherits from the FraxlendPairCore contract can be changed through a call to the transferOwnership function. This function internally calls the _setOwner function, which immediately sets the contract’s new owner. Making such a critical change in a single step is error-prone and can lead to irrevocable mistakes.

**Exploit Scenario**

Alice, a Frax Finance administrator, invokes the transferOwnership function to change the address of an existing contract’s owner but mistakenly submits the wrong address. As a result, ownership of the contract is permanently lost.

**Recommendations**

Short term, implement ownership transfer operations that are executed in a two-step process, in which the owner proposes a new address and the transfer is completed once the new address has executed a call to accept the role.

Long term, identify and document all possible actions that can be taken by privileged accounts and their associated risks. This will facilitate reviews of the codebase and prevent future mistakes.

## Missing checks of constructor/initialization parameters

**Description**

In the Fraxlend protocol’s constructor function, various settings are configured; however, two of the configuration parameters do not have checks to validate the values that they are set to.

First, the _liquidationFee parameter does not have an upper limit check:

Second, the Fraxlend system can work with one or two oracles; however, there is no check to ensure that at least one oracle is set:

**Exploit Scenario**

Bob deploys a custom pair with a misconfigured _configData argument in which no oracle is set. As a consequence, the exchange rate is incorrect.

**Recommendations**

Short term, add an upper limit check for the _liquidationFee parameter, and add a check for the _configData parameter to ensure that at least one oracle is set. The checks can be added in either the FraxlendPairCore contract or the FraxlendPairDeployer contract.

Long term, add appropriate requirements to values that users set to decrease the likelihood of user error.

## Improper validation of Chainlink data

**Description**

The current validation of the values returned by Chainlink’s latestRoundData function could result in the use of stale data.

The latestRoundData function returns the following values: the answer, the roundId (which represents the current round), the answeredInRound value (which corresponds to the round in which the answer was computed), and the updatedAt value (which is the timestamp of when the round was updated). An updatedAt value of zero means that the round is not complete and should not be used. An answeredInRound value that is less than the roundId could indicate stale data.

However, the _updateExchangeRate function does not check for these conditions.

**Exploit Scenario**

Chainlink is not updated correctly in the current round, and Eve, who should be liquidated with the real collateral asset price, is not liquidated because the price reported is outdated and is higher than it is in reality.

**Recommendations**

Short term, have _updateExchangeRate perform the following sanity check: require(updatedAt != 0 && answeredInRound == roundId). This check will ensure that the round has finished and that the pricing data is from the current round. Long term, when integrating with third-party protocols, make sure to accurately read their documentation and implement the appropriate sanity checks.

## Unapproved lenders could receive fTokens

**Description**

A Fraxlend custom pair can include a list of approved lenders; these are the only lenders who can deposit the underlying asset into the given pair and receive the corresponding fTokens. However, the system does not perform checks when users transfer fTokens; as a result, approved lenders could send fTokens to unapproved addresses.

Although unapproved addresses can only redeem fTokens sent to them—meaning this issue is not security-critical—the ability for approved lenders to send fTokens to unapproved addresses conflicts with the currently documented behavior.

**Exploit Scenario**

Bob, an approved lender, deposits 100 asset tokens and receives 90 fTokens. He then sends the fTokens to an unapproved address, causing other users to worry about the state of the protocol.

**Recommendations**

Short term, override the _beforeTokenTransfer function by applying the approvedLender modifier to it. Alternatively, document the ability for approved lenders to send fTokens to unapproved addresses.

Long term, when applying access controls to token owners, make sure to evaluate all the possible ways in which a token can be transferred and document the expected behavior.

## Missing checks in setter functions

**Description**

The setFee and setMinWaitPeriods functions do not have appropriate checks.

First, the setFee function does not have an upper limit, which means that the Fraxferry owner can set enormous fees. Second, the setMinWaitPeriods function does not require the new value to be at least one hour. A minimum waiting time of less than one hour would invalidate important safety assumptions. For example, in the event of a reorganization on the source chain, the minimum one-hour waiting time ensures that only transactions after the reorganization are ferried (as described in the code comment in figure 9.1).

**Exploit Scenario**

Bob, Fraxferry’s owner, calls setMinWaitPeriods with a _MIN_WAIT_PERIOD_ADD value lower than 3,600 (one hour), invalidating the waiting period’s protection regarding chain reorganizations.

**Recommendations**

Short term, add an upper limit check to the setFee function; add a check to the setMinWaitPeriods function to ensure that _MIN_WAIT_PERIOD_ADD and _MIN_WAIT_PERIOD_EXECUTE are at least 3,600 (one hour).

Long term, make sure that configuration variables can be set only to valid values.

## Risk of invalid batches due to unsafe cast in depart function

**Description**

The depart function performs an unsafe cast operation that could result in an invalid batch.

Users who want to send tokens to a certain chain use the various embark* functions. These functions eventually call embarkWithRecipient, which adds the relevant transactions to the transactions array.

At a certain point, the captain role calls depart with the start and end indices within transactions to specify the transactions inside of a batch. However, the depart function performs an unsafe cast operation when creating the new batch; because of this unsafe cast operation, an end value greater than 2 ** 64 would be cast to a value lower than the start value, breaking the invariant that end is greater than or equal to start.

If the resulting incorrect batch is not disputed by the crew member roles, which would cause the system to enter a paused state, the first officer role will call disembark to actually execute the transactions on the target chain. However, the disembark function’s third check, highlighted in figure 10.3, on the invalid transaction will fail, causing the transaction to revert and the system to stop working until the incorrect batch is removed with a call to removeBatches.

**Exploit Scenario**

Bob, Fraxferry’s captain, calls depart with an end value greater than 2 ** 64, which is cast to a value less than start. As a consequence, the system becomes unavailable either because the crew members called disputeBatch or because the disembark function reverts.

**Recommendations**

Short term, replace the unsafe cast operation in the depart function with a safe cast operation to ensure that the end >= start invariant holds.

Long term, implement robust unit tests and fuzz tests to check that important invariants hold.

## Transactions that were already executed can be canceled

**Description**

The Fraxferry contract’s owner can call the jettison or jettisonGroup functions to cancel a transaction or a series of transactions, respectively. However, these functions incorrectly use the executeIndex variable to determine whether the given transaction has already been executed. As a result, it is possible to cancel an already executed transaction.

The problem is that executeIndex tracks executed batches, not executed transactions. Because a batch can contain more than one transaction, the check in the _jettison function (figure 11.1) does not work correctly.

Note that canceling a transaction that has already been executed does not cancel its effects (i.e., the tokens were already sent to the receiver).

**Exploit Scenario**

Two batches of 10 transactions are executed; executeIndex is now 2. Bob, Fraxferry’s owner, calls jettison with an index value of 13 to cancel one of these transactions. The call to jettison should revert, but it is executed correctly. The emitted Cancelled event shows that a transaction that had already been executed was canceled, confusing the off-chain monitoring system.

**Recommendations**

Short term, use a different index in the jettison and jettisonGroup functions to track executed transactions. Long term, implement robust unit tests and fuzz tests to check that important invariants hold.

## Lack of contract existence check on low-level call

**Description**

The execute function includes a low-level call operation without a contract existence check; call operations return true even if the _to address is not a contract, so it is important to include contract existence checks alongside such operations.

**Exploit Scenario**

Bob, Fraxferry’s owner, calls execute with _to set to an address that should be a contract; however, the contract was self-destructed. Even though the contract at this address no longer exists, the operation still succeeds.

**Recommendations**

Short term, implement a contract existence check before the call operation in the execute function. If the call operation is expected to send ETH to an externally owned address, ensure that the check is performed only if the _data.length is not zero.

Long term, carefully review the Solidity documentation, especially the “Warnings” section.

## Lack of two-step process for contract ownership changes

**Description**

The setOwner() function is used to change the owner of the PnLFixedRate contract. Transferring ownership in one function call is error-prone and could result in irrevocable mistakes.

This issue can also be found in the following locations:
● contracts/pnl/PnL.sol:36-42
● contracts/strategy/ConvexStrategy.sol:447-453
● contracts/strategy/keeper/GStrategyGuard.sol:92-97
● contracts/strategy/stop-loss/StopLossLogic.sol:73-78

**Exploit Scenario**

The owner of the PnLFixedRate contract is a governance-controlled multisignature wallet. The community agrees to change the owner of the strategy, but the wrong address is mistakenly provided to its call to setOwner, permanently misconfiguring the system.

**Recommendations**

Short term, implement a two-step process to transfer contract ownership, in which the owner proposes a new address and then the new address executes a call to accept the role, completing the transfer.

Long term, review how critical operations are implemented across the codebase to make sure they are not error-prone.

## Non-zero token balances in the GRouter can be stolen

**Description**

A non-zero balance of 3CRV, DAI, USDC, or USDT in the router contract can be stolen by an attacker.

The GRouter contract is the entrypoint for deposits into a tranche and withdrawals out of a tranche. A deposit involves depositing a given number of a supported stablecoin (USDC, DAI, or USDT); converting the deposit, through a series of operations, into G3CRV, the protocol’s ERC4626-compatible vault token; and depositing the G3CRV into a tranche. Similarly, for withdrawals, the user burns their G3CRV that was in the tranche and, after a series of operations, receives back some amount of a supported stablecoin (figure 3.1).

However, notice that during withdrawals the amount of stableTokens that will be transferred back to the user is a function of the current stableToken balance of the contract (see the highlighted line in figure 3.1). In the expected case, the balance should be only the tokens received from the threePool.remove_liquidity_one_coin swap (see L450 in figure 3.1). However, a non-zero balance could also occur if a user airdrops some tokens or they transfer tokens by mistake instead of calling the expected deposit or withdraw functions. As long as the attacker has at least 1 wei of G3CRV to burn, they are capable of withdrawing the whole balance of stableToken from the contract, regardless of how much was received as part of the threePool swap. A similar situation can happen with deposits. A non-zero balance of G3CRV can be stolen as long as the attacker has at least 1 wei of either DAI, USDC, or USDT.

**Exploit Scenario**

Alice mistakenly sends a large amount of DAI to the GRouter contract instead of calling the deposit function. Eve notices that the GRouter contract has a non-zero balance of DAI and calls withdraw with a negligible balance of G3CRV. Eve is able to steal Alice's DAI at a very small cost.

**Recommendations**

Short term, consider using the difference between the contract’s pre- and post-balance of stableToken for withdrawals, and depositAmount for deposits, in order to ensure that only the newly received tokens are used for the operations.

Long term, create an external skim function that can be used to skim any excess tokens in the contract. Additionally, ensure that the user documentation highlights that users should not transfer tokens directly to the GRouter and should instead use the web interface or call the deposit and withdraw functions. Finally, ensure that token airdrops or unexpected transfers can only benefit the protocol.

## Stop loss primer cannot be deactivated

**Description**

The stop loss primer cannot be deactivated because the keeper contract uses the incorrect function to check whether or not the meta pool has become healthy again.

The stop loss primer is activated if the meta pool that is being used for yield becomes unhealthy. A meta pool is unhealthy if the price of the 3CRV token deviates from the expected price for a set amount of time. The primer can also be deactivated if, after it has been activated, the price of the token stabilizes back to a healthy value. Deactivating the primer is a critical feature because if the pool becomes healthy again, there is no reason to divest all of the strategy’s funds, take potential losses, and start all over again.

The GStrategyResolver contract, which is called by a Gelato keeper, will check to identify whether a primer can be deactivated. This is done via the taskStopStopLossPrimer function. The function will attempt to call the GStrategyGuard.endStopLoss function to see whether the primer can be deactivated (figure 7.1).

However, the GStrategyGuard contract does not have an endStopLoss function. Instead, it has a canEndStopLoss function. Note that the executor variable in taskStopStopLossPrimer is expected to implement the IGStrategyGuard function, which does have an endStopLoss function. However, the GStrategyGuard contract implements the IGuard interface, which does not have the endStopLoss function. Thus, the call to endStopLoss will simply return, which is equivalent to returning false, and the primer will not be deactivated.

**Exploit Scenario**

Due to market conditions, the price of the 3CRV token drops significantly for an extended period of time. This triggers the Gelato keeper to activate the stop loss primer. Soon after, the price of the 3CRV token restabilizes. However, because of the incorrect function call in the taskStopStopLossPrimer function, the primer cannot be deactivated, the stop loss process completes, and all the funds in the strategy must be divested.

**Recommendations**

Short term, change the function call from endStopLoss to canEndStopLoss in taskStopStopLossPrimer.

Long term, ensure that there are no near-duplicate interfaces for a given contract in the protocol that may lead to an edge case similar to this. Additionally, expand the unit test suite to cover additional edge cases and to ensure that the system behaves as expected.

## getYieldTokenAmount uses convertToAssets instead of convertToShares

**Description**

The getYieldTokenAmount function does not properly convert a 3CRV token amount into a G3CRV token amount, which may allow a user to withdraw more or less than expected or lead to imbalanced tranches after a migration.

The expected behavior of the getYieldTokenAmount function is to return the number of G3CRV tokens represented by a given 3CRV amount. For withdrawals, this will determine how many G3CRV tokens should be returned back to the GRouter contract. For migrations, the function is used to figure out how many G3CRV tokens should be allocated to the senior and junior tranches.

To convert a given amount of 3CRV to G3CRV, the GVault.convertToShares function should be used. However, the getYieldTokenAmount function uses the GVault.convertToAssets function (figure 8.1). Thus, getYieldTokenAmount takes an amount of 3CRV tokens and treats it as shares in the GVault, instead of assets.

If the system is profitable, each G3CRV share should be worth more over time. Thus, getYieldTokenAmount will return a value larger than expected because one share is worth more than one asset. This allows a user to withdraw more from the GTranche contract than they should be able to. Additionally, a profitable system will cause the senior tranche to receive more G3CRV tokens than expected during migrations. A similar situation can happen if the system is not profitable.

**Exploit Scenario**

Alice deposits $100 worth of USDC into the system. After a certain amount of time, the GSquared protocol becomes profitable and Alice should be able to withdraw $110, making $10 in profit. However, due to the incorrect arithmetic performed in the getYieldTokenAmount function, Alice is able to withdraw $120 of USDC.

**Recommendations**

Short term, use convertToShares instead of convertToAssets in getYieldTokenAmount.

Long term, expand the unit test suite to cover additional edge cases and to ensure that the system behaves as expected.

## convertToShares can be manipulated to block deposits

**Description**

An attacker can block operations by using direct token transfers to manipulate convertToShares, which computes the amount of shares to deposit.

convertToShares is used in the GVault code to know how many shares correspond to certain amount of assets

This function relies on the _freeFunds function to calculate the amount of shares

In the simplest case, _calculateLockedProfit() can be assumed as zero if there is no locked profit. 

However, the fact that _totalAssets has a lower bound determined by asset.balanceOf(address(this)) can be exploited to manipulate the result by "donating" assets to the GVault address.

**Exploit Scenario**

Alice deploys a new GVault. Eve observes the deployment and quickly transfers an amount of tokens to the GVault address. One of two scenarios can happen:
1. Eve transfers a minimal amount of tokens, forcing a positive amount of freeFunds. This will block any immediate calls to deposit, since it will result in zero shares to be minted.
2. Eve transfers a large amount of tokens, forcing future deposits to be more expensive or resulting in zero shares. Every new deposit can increase the amount of free funds, making the effect more severe.

It is important to note that although Alice cannot use the deposit function, she can still call mint to bypass the exploit.

**Recommendations**

Short term, use a state variable, assetBalance, to track the total balance of assets in the contract. Avoid using balanceOf, which is prone to manipulation.

Long term, expand the unit test suite to cover additional edge cases and to ensure that the system behaves as expected.

## Incorrect rounding direction in GVault

**Description**

The minting and withdrawal operations in the GVault use rounding in favor of the user instead of the protocol, giving away a small amount of shares or assets that can accumulate over time.

convertToShares is used in the GVault code to know how many shares correspond to a certain amount of assets:

This function rounds down, providing slightly fewer shares than expected for some amount of assets.

Additionally, convertToAssets is used in the GVault code to know how many assets correspond to certain amount of shares:

This function also rounds down, providing slightly fewer assets than expected for some amount of shares.

However, the mint function uses previewMint, which uses convertToAssets:

This means that the function favors the user, since they get some fixed amount of shares for a rounded-down amount of assets.

In a similar way, the withdraw function uses convertToShares:

This means that the function favors the user, since they get some fixed amount of assets for a rounded-down amount of shares.

This issue should also be also considered when minting fees, since they should favor the protocol instead of the user or the strategy.

**Exploit Scenario**

Alice deploys a new GVault and provides some liquidity. Eve uses mints and withdrawals to slowly drain the liquidity, possibly affecting the internal bookkeeping of the GVault.

**Recommendations**

Short term, consider refactoring the GVault code to specify the rounding direction across the codebase in order keep the error in favor of the user or the protocol.

Long term, expand the unit test suite to cover additional edge cases and to ensure that the system behaves as expected.

## Incorrect slippage calculation performed during strategy investments and divestitures

**Description**

The incorrect arithmetic calculation for slippage tolerance during strategy investments and divestitures can lead to an increased rate of failed profit-and-loss (PnL) reports and withdrawals.

The ConvexStrategy contract is tasked with investing excess funds into a meta pool to obtain yield and divesting those funds from the pool whenever necessary. Investments are done via the invest function, and divestitures for a given amount are done via the divest function. Both functions have the ability to manage the amount of slippage that is allowed during the deposit and withdrawal from the meta pool. For example, in the divest function, the withdrawal will go through only if the amount of 3CRV tokens that will be transferred out from the pool (by burning meta pool tokens) is greater than or equal to the _debt, the amount of 3CRV that needs to be transferred out from the pool, discounted by baseSlippage (figure 13.1). Thus, both sides of the comparison must have units of 3CRV.

To calculate the value of a meta pool token (mpLP) in terms of 3CRV, the curveValue function is called (figure 13.2). The units of the return value, ratio, are 3CRV/mpLP.

However, note that in figure 13.1, meta_amount value, which is the amount of mpLP tokens that need to be burned, is divided by ratio. From a unit perspective, this is multiplying an mpLP amount by a mpLP/3CRV ratio. The resultant units are not 3CRV. Instead, the arithmetic should be meta_amount multiplied by ratio. This would be mpLP times 3CRV/mpLP, which would result in the final units of 3CRV.

Assuming 3CRV/mpLP is greater than one, the division instead of multiplication will result in a smaller value, which increases the likelihood that the slippage tolerance is not met. The invest and divest functions are called during PnL reporting and withdrawals. If there is a higher risk for the functions to revert because the slippage tolerance is not met, the likelihood of failed PnL reports and withdrawals also increases.

**Exploit Scenario**

Alice wishes to withdraw some funds from the GSquared protocol. She calls GRouter.withdraw and with a reasonable minAmount. The GVault contract calls the ConvexStrategy contract to withdraw some funds to meet the necessary withdrawal amount. The strategy attempts to divest the necessary amount of funds. However, due to the incorrect slippage arithmetic, the divest function reverts and Alice’s withdrawal is unsuccessful.

**Recommendations**

Short term, in divest, multiply meta_amount by ratio. In invest, multiply amount by ratio.

Long term, expand the unit test suite to cover additional edge cases and to ensure that the system behaves as expected.

## Potential division by zero in _calcTrancheValue

**Description**

Junior tranche withdrawals may fail due to an unexpected division by zero error.

One of the key steps performed during junior tranche withdrawals is to identify the dollar value of the tranche tokens that will be burned by calling _calcTrancheValue (figure 14.1)

To calculate the dollar value, the factor function is called to identify how many tokens represent one dollar. The dollar value, amount, is then the token amount provided, _amount, divided by factor.

However, an edge case in the factor function will occur if the total supply of tranche tokens (junior or senior) is non-zero while the amount of assets backing those tokens is zero. Practically, this can happen only if the system is exposed to a loss large enough that the assets backing the junior tranche tokens are completely wiped. In this edge case, the factor function returns zero (figure 14.2). The subsequent division by zero in _calcTrancheValue will cause the transaction to revert.

It is important to note that if the system enters a state where there are no assets backing the junior tranche, junior tranche token holders would be unable to withdraw anyway. However, this division by zero should be caught in _calcTrancheValue, and the requisite error code should be thrown.

**Recommendations**

Short term, add a check before the division to ensure that factor is greater than zero. If factor is zero, throw a custom error code specifically created for this situation.

Long term, expand the unit test suite to cover additional edge cases and to ensure that the system behaves as expected.

## Token withdrawals from GTranche are sent to the incorrect address

**Description**

The GTranche withdrawal function takes in a _recipient address to send the G3CRV shares to, but instead sends those shares to msg.sender (figure 15.1).

Since GTranche withdrawals are performed by the GRouter contract on behalf of the user, the msg.sender and _recipient address are the same. However, a direct call to GTranche.withdraw by a user could lead to unexpected consequences.

**Recommendations**

Short term, change the destination address to _recipient instead of msg.sender.

Long term, increase unit test coverage to include tests directly on GTranche and associated contracts in addition to performing the unit tests through the GRouter contract.

## Risk of accounting errors due to missing check in the invest function

**Description**

Because of a missing check in the invest function, investing multiple tokens with different decimals in the same strategy will result in incorrect profit-and-loss (PnL) reporting, which could result in the loss of user or protocol funds.

The invest function is responsible for transferring funds from the treasury to a strategy and for updating the strategy’s investment balance (i.e., strategyInvestedAmount). However, the invest function accepts any token in the collateral array alongside the token amounts to be transferred. Therefore, if multiple tokens with different decimals are used to invest in the same strategy, the treasury’s investment records would not accurately reflect the true balance of the strategy, resulting in accounting errors within the protocol.

**Exploit Scenario**

The CompoundStrategy contract is supposed to accept only USDC as its investment token. However, the fund manager, Bob, triggers investments into the Compound strategy using FRAX tokens instead. Due to insufficient validation of the collateralToken value, the transaction succeeds, causing a mismatch between the treasury’s account and the total value of assets in the strategy.

**Recommendations**

Short term, implement a check within the invest function to ensure that the collateralToken address of the supplied collateralIndex value matches the token accepted by the strategy.

Long term, carefully review token usage across all contracts to ensure that each token’s decimal places are taken into consideration.

## Missing functionality in the _rescueTokens function

**Description**

The RegistryClient contract is a helper contract designed to aid in the protocol’s role-based access control (RBAC) mechanism. It has various helper functions that serve as safety mechanisms to rescue funds trapped inside a contract. The inline documentation for the _rescueTokens function states that if the _amounts array contains a zero-value entry, then the entire token’s balance should be transferred to the caller. However, this functionality is not present in the code; instead, the function sends zero tokens to the caller on a zero-value entry.

**Exploit Scenario**

A user accidentally sends ERC20 tokens to the PSM contract. To rescue the tokens, Bob, the guardian of the Ondo protocol contracts, calls _rescueTokens and specifies zero as the amount for the ERC20 tokens in question. However, the contract incorrectly transfers zero tokens to him and, therefore, the funds are not rescued.

**Recommendations**

Short term, adjust the _rescueTokens function so that it transfers the entire token balance when it reaches a zero-value entry in the _amounts array.

Long term, improve the unit test coverage to cover additional edge cases and to ensure that the system behaves as expected.

## Lack of contract existence check on call

**Description**

The factories and the registry client all have a multiexcall function, which is designed to create batched calls to various target addresses. To do this, it uses the call opcode to execute arbitrary calldata. If the target address is set to an incorrect address, the address of an externally owned account (EOA), or the address of a contract that is subsequently destroyed, a call to the target will still return true. However, the multiexcall functions in the factories and the registry client do not include contract existence checks to account for this behavior.

**Exploit Scenario**

Bob, the guardian of the Ondo protocol contracts, uses the multiexcall function but accidentally passes in an incorrect address that points to an EOA. As a result, the call succeeds without executing any code.

**Recommendations**

Short term, implement a contract existence check before each call. Document the fact that calling an EOA or self-destructed contract will still return true.

Long term, carefully review the Solidity documentation, especially the “Warnings” section, and the pitfalls of using low-level patterns.

## Problematic use of safeApprove

**Description**

In order for users to earn yield on the collateral tokens they deposit, the Treasury contract sends the collateral tokens to a yield-bearing strategy contract that inherits from the BaseStablecoinStrategy contract. When a privileged actor calls the redeem function, the function approves the Treasury contract to pull the necessary funds from the strategy.

However, the function approves the Treasury contract by calling the safeApprove function.

As explained in the OpenZeppelin documentation, safeApprove should be called only if the currently approved amount is zero—when setting an initial allowance or when resetting the allowance to zero. Therefore, if the entire approved amount is not pulled by calling Treasury.withdraw, subsequent redemption operations will revert.

Additionally, OpenZeppelin’s documentation indicates that the safeApprove function is officially deprecated.

**Exploit Scenario**

Bob, the fund manager of the Ondo protocol, deposits 200 tokens into the CompoundStrategy contract and calls redeem to redeem 100 of them. However, he calls Treasury.withdraw for only 50 tokens. Because the allowance allocated for safeApprove has not been completely used, he is blocked from performing subsequent redemption operations for the remaining 100 tokens.

**Recommendations**

Short term, use safeIncreaseAllowance and safeDecreaseAllowance instead of safeApprove. Alternatively, document the fact that the Treasury should use up the entire approval amount before the privileged actor can call a strategy’s redeem function.

Long term, expand the dynamic fuzz testing suite to identify edge cases like this.

## Lack of upper bound for fees and system parameters

**Description**

The PSM contract’s setMintFee and setRedeemFee functions, used by privileged actors to set optional minting and redeeming fees, do not have an upper bound on the fee amount that can be set; therefore, a privileged actor could set minting and redeeming fees to any value. Excessively high fees resulting from typos would likely not be noticed until they cause disruptions in the system.

Additionally, a large number of system parameters throughout the Rewarder, Treasury, PSM, and PolyMinter contracts are unbounded.

**Recommendations**

Short term, set upper limits for the minting and redeeming fees and for all the system parameters that do not currently have upper bounds. The upper bounds for the fees should be high enough to encompass any reasonable value but low enough to catch mistakenly or maliciously entered values that would result in unsustainably high fees.

Long term, carefully document the owner-specified values that dictate the financial properties of the contracts and ensure that they are properly constrained.

## Ability to create wallets before WalletFactory is initialized

**Description**

When a user creates a new wallet through the WalletFactory contract, a check for whether the factory has been initialized correctly is missing. This missing check can result in the creation of unusable wallets.

When a new wallet is created, the _deployWallet function is called, which deploys a proxy contract given the bytecode from the _bytecodeWithArgs function.

If the _deployWallet function is called but the WalletFactory contract has not been yet initialized, then the proxyByteCode variable will be empty. And if the bytecode resulting from the abi.encodePacked(implementationResolver, bytes32("NestedWallet")) function can be interpreted as valid executable bytecode, then the wallet creation will succeed, creating a nonfunctioning wallet.

The above situation can happen when the implementation resolver address starts with a certain value, such as 0x00, which is interpreted as the stop opcode.

As noted by Nested Finance, this issue is currently not reachable due to the use of the reentrancy modifier. See the TOB-NESTED-16 fix review results in the fix review appendix for more details.

**Exploit Scenario**

Alice creates a new NestedWallet proxy. The contract creation succeeds; however, Alice is unable to use her wallet because the contract deployer forgot to initialize WalletFactory.

**Recommendations**

Short term, have _deployWallet check whether initialized is true or require that the proxy bytecode be non-empty. 

Long term, create diagrams or flowcharts on the wallet creation process, including the creation of WalletFactory. Document any requirements to be fulfilled for any function, and ensure that the relevant functions revert if those conditions are not met.

## Risk of portfolio voting power theft

**Description**

Due to incorrect bookkeeping, anyone can steal a portfolio’s voting power.

The deposit function deposits the given number of the user’s NST (amount) and increases the chosen portfolio’s power by that number.

The withdraw function does the opposite—it withdraws the user’s deposit and decreases the chosen portfolio’s power by the number of tokens withdrawn.

However, there is no check to ensure that the portfolio used to withdraw was the one used to deposit. As a result, an attacker could deposit NST to increase the power of portfolio A and withdraw NST to decrease the power of portfolio B.

**Exploit Scenario**

Bob delegates 100 NST of bribes to Alice's portfolio. Eve allocates 100 NST of bribes to herself. Eve then removes 100 NST of bribes from Alice's portfolio. As a result, Eve has stolen 100 NST of voting power from Alice’s portfolio.

**Recommendations**

Short term, use mapping(address depositor => address portfolio) to track portfolio bookkeeping.

Long term, carefully document the data structure choices and their assumptions. Ensure that proper validation and tests are placed on operations that impact and are impacted by user funds. Consider using Echidna to test multiple-transaction invariants on the Nested contracts.

## acceptDeal adds unreleased tokens to new vesting schedules

**Description**

Claimable tokens become unclaimable if new vesting schedules are created, delaying the opportunity for users to withdraw.

When a deal is accepted through the acceptDeal function during an active vesting schedule, a new vesting schedule is automatically pushed.

However, claimable tokens in the previous schedule will be added to the new vesting schedule instead of being released. This creates a delayed vesting schedule and a slower token release than expected.

**Exploit Scenario**

Bob receives 1,001 NST that will be vested over the next two years. After one year, Bob creates an OTC deal for 1 NST. Bob still has not claimed any of his claimable NST tokens (around 500 NST). Alice, the owner, accepts Bob’s deal, and Bob receives 1 vested NST. Because his deal was accepted, a new vesting schedule is automatically created for Bob. The new schedule is set to release all of his remaining 1,000 NST over the remaining year. Now Bob is temporarily unable to claim his originally claimable 500 NST.

**Recommendations**

Short term, include a call to _releaseNst in acceptDeal so that all claimable tokens are released before a new vesting schedule is pushed.

Long term, create a diagram highlighting the state transitions of the Vesting contract. In this diagram, emphasize the transition to new vesting schedules and identify underlying invariants. Ensure that unit tests cover every transition. Consider using Echidna to test invariants related to vesting.

## Unbounded loops could cause denial of service for third-party integrations

**Description**

Unbounded loops to compute the total number of tokens released in the VestingFactory and Vesting contracts can result in a denial of service of the contracts integrating with these Vesting contracts.

The VestingFactory contract creates Vesting contracts and stores their addresses in the vestings variable. The totalReleased function in the VestingFactory contract iterates over this list of vesting contracts and calls itself on the given Vesting contract to compute the total number of tokens released to all of the beneficiaries.

The totalReleased function in the Vesting contract also iterates over the list of schedules to compute the total number of tokens released to the beneficiaries. A new schedule is added to the list for every deal made between the start and end of the vesting period. This way, the number of schedules can grow to any arbitrary number.

Therefore, the totalReleased function in the VestingFactory contract can throw an out-of-gas error if too many vestings or schedules are added, resulting in a denial of service of the contract calling this function.

Similar unbounded loops are used in other functions like totalReleasable and totalAllocation in the VestingFactory contract and the totalDealed function in the Vesting contract. All of these functions can also cause a denial of service of the contract calling them in a transaction.

**Exploit Scenario**

The Nested Finance team creates 100 Vesting contracts, one for each team member and stakeholder. Eve, one of the team members, creates 100,000 deals to sell her tokens during the vesting period. The owner accepts all of these deals. Now, any contract calling the totalReleased function on the VestingFactory contract will need to iterate through 10,000,000 items, which may result in an out-of-gas error and make the contract unusable.

**Recommendations**

Short term, document this behavior to make sure third parties are aware of the risks.

Long term, carefully review operations that consume a large amount of gas, especially those in loops. Model any variable-length loops to ensure that they cannot block contract execution within the expected system parameter limits.

## Risk of unlimited slippage in NestedDca swaps

**Description**

The swaps executed by the NestedDca contract from Uniswap V3 allow unlimited slippage because the slippage protection is assigned to a memory variable that is not passed on to the next function executing the swap.

When executing swaps on Uniswap, a slippage check parameter is required to protect users from paying an unexpectedly higher amount for the token being bought.

The NestedDca contract executes the DCA by swapping user-configured tokens in a loop, as shown below

The swap parameters are assigned from contract storage to a local memory variable named swapParams. The slippage value is computed and set to the memory variable swapParams.amountOutMinimum. This value is not set to the storage variable; therefore, it is visible only in the scope of this function, not in other functions that read swap parameters from storage.

The performDca function then calls the _performDcaSwap function, which again assigns swap parameters from contract storage to a local memory variable named swapParams, as shown below

So the value of swapParams.amountOutMinimum is not correct in the scope of the _performDcaSwap function—it is 0, indicating no slippage protection. This incorrect value is then sent as an argument to the Uniswap V3 router, which executes the swap without slippage protection. An attacker can front-run or sandwich the swap transaction to benefit from this unprotected swap transaction.

**Exploit Scenario**

Alice uses a DCA strategy to buy $50,000 worth of MyToken every day. The DCA is executed every day at 9 a.m. ET by the Gelato operators. Eve front-runs the transaction, buys $10,000 worth of MyToken, and sells them directly after Alice's transaction. As a result, Alice bought MyToken at an unfair price, and Eve profited from it.

**Recommendations**

Short term, have the code pass the value of the memory variable swapParams from the performDca function to the _performDcaSwap function as an argument.

Long term, carefully review the code to check for the correct use of values set to memory variables. Add unit test cases that mimic front-running to capture such issues.

## Insufficient DCA existence check could result in an inconsistent state

**Description**

After renouncing ownership of a DCA, a user can recreate the same DCA and lose access to the tasks associated with the previous DCA. This can break assumptions made by Gelato’s components and break other third-party integrations

The createDca function computes an ID for a DCA by hashing the parameters provided by the user. It then checks for the existence of a DCA with the same ID by checking whether an owner is already assigned to the given dcaId.

The renounceDcaOwnership function allows a user to renounce ownership of a DCA by setting the owner to address 0x0. However, this function does not check whether the Gelato task created for the DCA has been stopped.

If a user who renounced ownership creates a new DCA with the same parameters, the new DCA will have the same dcaId as the previous one and will reset the isGelatoWatching and taskId variables.

Previously created tasks will no longer be associated with the DCA in the Nested contract, preventing them from being stopped. They will still be enabled in the Gelato network. As a result, multiple unstoppable tasks can be created for the same DCA. This could break the system’s integrations with third-party components

**Exploit Scenario**

Alice creates and starts a DCA. After some time, she renounces her ownership of it. Later, Alice creates a new DCA with exactly the same parameters. A new DCA gets created with her previous DCA’s dcaId, which already has a Gelato task running for it. Alice starts the new DCA again and selects a different feeToken, allowing her to create a new Gelato task for her DCA, which will then set a new the dca.taskId in the contract storage. This leaves Alice with two active Gelato tasks for one dcaId.

**Recommendations**

Short term, implement our recommendations for an alternative schema for storing DCAs in the NestedDca contract, proposed in appendix C.

Long term, document the expected validation that should occur when a DCA is created. Create a diagram highlighting the life cycle of a DCA and the underlying invariants. Create unit tests for each state transition, and consider using Echidna to test multiple-transaction invariants.

## Users prevented from using special ETH address to pay Gelato fees

**Description**

The startDca function’s check of the provided fee token address blocks users from using ETH to pay Gelato fees. 

The startDca function allows users to start their DCAs using a particular fee token address to pay Gelato fees. As part of the validation, it checks that the fee token address points to a deployed address (i.e., an address with deployed code).

Gelato fees are paid by the _transferGelatoFees function in the token specified by the user

However, the special ETH address (0xEee..) will never have deployed code; therefore, it cannot be used as a fee token. If 0xEee is supplied, the else branch in _transferGelatoFees is unreachable

**Exploit Scenario**

Alice creates a DCA and wants to pay the Gelato fees using ETH. She checks the Gelato documentation and uses the correct fee token address, but the NestedDca contract reverts when she tries to start her DCA. She is forced to use another token to pay her fees.

**Recommendations**

Short term, properly document the special ETH address and modify the startDca function to allow users to use the ETH address to pay Gelato fees.

Long term, make sure that the code has enough unit and integration tests to test all of its features.

## DCA tasks can be stopped by anyone


**Description**

Stopping an active DCA task relies on a user-provided parameter, the task ID. However, there is no validation performed on this value, so any automated DCA task can be stopped by anyone.

The stopDca function takes as input a DCA ID and a gelato-ops task ID.

Ownership of the DCA is verified through the onlyDcaOwner(dcaId) modifier. However, there are no checks performed on the _taskId parameter to verify that the given task ID belongs to the provided DCA ID (i.e., require(_taskId == dcas[dcaId].taskId)), or that the given DCA has ever successfully started a task in the first place.

This means that anyone is able to supply a task ID of a running DCA to shut down another user's automated task.

**Exploit Scenario**

Alice sets up a task to dollar-cost-average her 100 million TokenA back to USDC over the next month. Eve stops Alice’s DCA task. After one month, TokenA's value suddenly declines to nearly zero. Without her knowledge, Alice’s token has lost all of its value, since the DCA task was not performed after being stopped by Eve without Alice’s approval.

**Recommendations**

Short term, add a check that verifies that the task ID matches the task ID from the provided DCA, or have the code simply directly access the task ID stored for the given DCA.

Long term, document the expected validation that should occur when a DCA task is stopped. Create a diagram highlighting the life cycle of a DCA and the underlying invariants. Create unit tests for each state transition, and consider using Echidna to test multiple-transaction invariants.

## DCA task IDs can be manually set

**Description**

The taskId parameter of a DCA should be set only by Gelato, but it can be arbitrarily set by the user. This can break assumptions made by Gelato’s components and break other third-party integrations.

The createDca function does not check whether the user has manually set the dca.taskId parameter, even though it verifies that other parameters have not been manually set.

The task ID should be assigned only by Gelato when startDca is called. It can be assigned only once and is meant to be unique to one created DCA task.

Because the code does not ensure that task IDs have been set only by Gelato, an attacker could claim to be the creator of any existing task or of a task that does not exist. This could break the system’s integration with third-party components.

**Exploit Scenario**

Alice has a bot that watches the task ID of every DCA. Bob creates a DCA and a task ID of 0x41414141. Eve creates a new DCA and sets the task ID to 0x41414141. Alice’s bot was not designed to handle multiple DCAs with the same task ID, so it crashes.

**Recommendations**

Short term, implement our recommendations for an alternative schema for storing DCAs in the NestedDca contract, proposed in appendix C.

Long term, document the expected validation that should occur when a DCA is created. Create a diagram highlighting the life cycle of a DCA and the underlying invariants. Create unit tests for each state transition, and consider using Echidna to test multiple-transaction invariants.

## restartDca lacks data validation

**Description**

The restartDca function has similar functionality to the createDca function; however, it lacks important safety checks and could be abused.

restartDca behaves similarly to createDca. It sets up a DCA and stores its parameters and owner.

The following are the steps in which restartDca differs from createDca:
1. It does not check whether the .isGelatoWatching parameter is already set.
2. It does not check whether the new DCA already exists and is running.
3. It does not set the maximum token allowance for the relevant tokens.
4. It does not check whether dca.swapsParams[i].sqrtPriceLimitX96 is 0.

The missing check in step 1 allows users to set the .isGelatoWatching and .taskId system parameters. Any user could stop any DCA task by resetting an existing DCA, spoofing the task ID, and setting isGelatoWatching to true.

The missing check in step 2 allows users to overwrite an existing DCA. Without this check, users can disrupt an existing DCA by setting isGelatoWatching to false again or taskId to the empty bytes32 value. By resetting the taskId, the original task will not be associated with a DCA anymore, and it will not be possible to cancel it.

Furthermore, the missing approvals in step 3 mean that the performDca function will fail if the tokens for the new DCA differ and if the NestedDca contract has not previously given token spending approval for these tokens.

**Exploit Scenario**

Alice sets up a task to dollar-cost-average her 100 million TokenA back to USDC over the next month. Eve stops Alice’s DCA task. After one month, TokenA's value suddenly declines to nearly zero. Without her knowledge, Alice’s token has lost all of its value, since the DCA was not performed after being stopped by Eve without Alice’s approval.

**Recommendations**

Short term, implement the four missing steps described in this finding in restartDca.

Long term, document the expected validation that should occur when a DCA is created. Create a diagram highlighting the life cycle of a DCA and the underlying invariants. Create unit tests for each state transition, and consider using Echidna to test multiple-transaction invariants.

## Reentrancy vulnerabilities in NestedDca could allow Gelato operators to drain leftover tokens

**Description**

The performDca and updateDcaAmount functions contain a reentrancy vulnerability that could allow malicious Gelato operators to drain leftover tokens.

The performDca function lets Gelato operators perform DCA tasks. The safeTransferFrom function is called before _performDcaSwap to transfer the number of tokens specified in the swapParams.amountIn variable

The _performDcaSwap function calls the Uniswap router and swaps the number of tokens specified in swapParams.amountIn

The DCA’s owner can change the value of swapParams.amountIn at any time by calling the updateDcaAmount function

However, there is no reentrancy protection in either performDca or updateDcaAmount. If a token being transferred has a callback mechanism, a malicious Gelato operator could do the following:
● Make performDca transfer a small number of tokens
● Reenter into updateDcaAmount to change swapParams.amountIn to be a larger number
● Trigger the Uniswap swap with more tokens than were sent through the DCA task

As a result, a malicious Gelato operator could steal tokens that are left in the contract.

**Exploit Scenario**

Eve is a malicious Gelato operator. She notices that $10,000 worth of TokenA are stuck in the NestedDca contract. TokenA has a callback mechanism. Eve exploits the reentrancy vulnerability in performDca to steal the tokens.

**Recommendations**

Short term, add the nonReentrant modifier to performDca and updateDcaAmount.

Long term, carefully evaluate every function that does not follow the check-effect-interaction pattern. Document any known reentrancy risks.

## Borrower can drain lender assets by withdrawing the cancellationFee multiple times

**Description**

If a loan product is canceled by governance and the cancellation fee that the borrower is able to withdraw is non-zero, the borrower can drain the assets deposited by lenders by withdrawing his cancellation fee multiple times.

The protocol requires borrowers to deposit a cancellation fee equal to some percentage of the target amount of tokens the borrower would like to be able to borrow. If the loan is canceled, the borrower may be able to withdraw either the full cancellation fee or a part of the cancellation fee that was not distributed to lenders. The exact amount is defined by the cancellationFeeEscrow state variable, and the withdrawal can be done using the withdrawRemainingEscrow function.

This will transfer the assets out of the PoolCustodian contract, which holds the borrower's deposit as well as the lender's deposits, to the borrower-provided address.

Although the total deposited amount is reduced correctly in the PoolCustodian contract, the cancellationFeeEscrow state variable is never reduced to zero. This allows the borrower to withdraw the same amount of tokens multiple times, draining the pool of all assets deposited by the lenders.

**Exploit Scenario**

A bullet loan is created for Bob as the borrower with a target origination amount of 10,000 USDC. Bob deposits a 1,000 USDC cancellation fee into the pool, which begins the book-building phase of the pool.

Alice deposits 10,000 USDC into the pool at a 5% rate. After some time has passed, the governance transitions the bullet loan into the origination phase so Bob can borrow the asset.

Bob decides not to borrow from the pool and the loan is canceled by governance, which decides not to redistribute the cancellation fee to Alice.

Bob has 1,000 USDC in escrow and can withdraw it using the withdrawRemainingEscrow function. Since the value of the cancellationFeeEscrow state variable is never reduced, Bob can withdraw his cancellation fee nine more times, draining Alice's deposit.

**Recommendations**

Short term, set the cancellationFeeEscrow state variable to zero when the borrower withdraws their cancellation fee.

Long term, implement additional unit tests that test for edge cases. Come up with a list of system and function-level invariants and test them with Echidna.

## Incorrect fee calculation on withdrawal can lead to DoS of withdrawals or loss of assets

**Description**

Incorrect fee calculation in the withdraw function of the LoanLender contract can lead to inflated fees if the asset used does not have 18 decimals of precision. This will prevent any lender from withdrawing their deposit and earnings, or potentially make the lenders lose a significant part of their deposit and earnings.

The protocol allows lenders to withdraw their share of earnings once a borrower has repaid a part of their borrowed amount. These earnings are reduced by a small protocol fee in the withdraw function, shown in figure 2.1. The calculation uses WAD (shown in figure 2.2) as a scaling factor since the FEES_CONTROLLER uses 18 decimals of precision, while the LoanLender contract uses the same decimal precision as the token.

However, due to an incorrect calculation, these fees will be several orders of magnitude higher (or lower) than intended. Depending on the decimal precision of the token used, this can lead to three different scenarios:
1. If the token decimals are sufficiently smaller than 18, this will stop the function from correctly executing, preventing any lenders from withdrawing their deposit and earnings.
2. If the token decimals are slightly smaller than 18, the protocol will charge a higher-than-intended fee, making the lenders lose a significant amount of their earnings.
3. If the token decimals are higher than 18, the protocol will charge a smaller-than-intended fee, which will lead to a loss for the protocol.

**Exploit Scenario**

A coupon bullet loan is created for Alice with a target origination amount of 1,000,000 USDC and a repayment fee of 1% (0.01e18). Bob deposits 1,000,000 USDC at a 10% rate so it is available for Alice to borrow.

Alice borrows the entire amount. Since this is a coupon loan, Alice will be repaying a share of the loan interest at a regular interval.

Assume Alice has repaid 1,000 USDC in her first coupon repayment and Bob's total earnings are 1000 USDC. Bob attempts to withdraw his earnings, but the function reverts due to an arithmetic underflow.

If we calculate the fee using the current calculation—keeping in mind that the fixed point calculation is set to the amount of decimals of USDC (i.e., six)—we get the following:

fee = 1000e6.mul(1e18).mul(0.01e18)

fee = 10000000000000000000000000e6 or 10,000,000,000,000,000,000,000,000 USDC

This fee far exceeds the deposited and earned amount, causing the function to revert due to an arithmetic underflow.

**Recommendations**

Short term, revise the calculation such that token amounts are properly normalized before using fixed point multiplication with the repayment fee. Consider setting the PoolTokenFeesController precision to be equal to the amount of token decimals.

Long term, replicate real-world system settings in unit tests with a variety of different tokens that have more (or fewer) than 18 decimals of precision. Set up fuzzing tests with Echidna for all calculations done within the system to ensure they work with different token decimals and hold proper precision.

## Lack of zero-address checks

**Description**

A number of functions in the codebase do not revert if the zero address is passed in for a parameter that should not be set to zero.

The following parameters, among others, do not have zero-address checks:
● The token and rolesManager variables in the PoolCustodian’s constructor
● The _rolesManager, _pool, and _rewardsOperator in updateRolesManager, initializePool, and updateRewardsOperator, respectively
● The factoryRegistry in LoanFactoryBase and in the RCLFactory
● The rolesManager in Managed.sol’s constructor
● The module address in RewardsManager.sol’s addModule function

**Exploit Scenario**

Alice, the deployer of the RCLFactory, accidentally sets the factoryRegistry parameter to zero in the constructor. As a result, the deploy function will always revert and the contract will be unusable.

The governance in control of the RewardsManager contract accidentally adds an invalid module via the addModule function. Since all user actions loop through all of the added modules, all user-staked positions become permanently locked, and users are prevented from unstaking or claiming rewards.

**Recommendations**

Short term, add zero-address checks for the parameters listed above and for all other parameters for which the zero address is not an acceptable value. Add a supportsInterface check for any modules added to the RewardsManager contract to ensure an invalid module cannot be added.

Long term, review input validation across components. Avoid relying solely on the validation performed by front-end code, scripts, or other contracts, as a bug in any of those components could prevent them from performing that validation.

## Problematic approach to data validation

**Description**

The Atlendis Labs infrastructure relies on validation functions that exist outside of contract constructors. This requires an explicit call to a validate helper function, which can result in unexpected behavior if a function forgets to call it prior to construction.

For example, the LoanState constructor saves a totalPaymentsCount variable, meant to represent the total number of payments for the loan, denominated by the LOAN_DURATION and PAYMENT_PERIOD. This means that this calculation can potentially round down to zero, but the constructor lacks checks to ensure it is non-zero.

Instead, the checks occur in the LoanFactoryBase contract, when deployLoan is called. This flow assumes that the helper function will be used.

This pattern is duplicated across the codebase in the form of helper functions, which are used in other contracts. As a result, the code relies on assumptions of prior validation elsewhere in the system. This practice is error-prone, as the validation checks are not guaranteed to run.

**Exploit Scenario**

Alice, an Atlendis protocol developer, changes the logic to deploy a loan. Because the constructor of the LoanState contract has no data validation checks, she makes an erroneous assumption about what is validated in another part of the system.

**Recommendations**

Short term, move the validation logic to the constructor of the LoanState contract instead of using the factory deployment.

Long term, always keep the validation logic as close as possible to locations in the code where variables are being used and declared. This ensures all the inputs are properly validated.

## Borrower can skip the last coupon payment

**Description**

If a borrower calls repay at the exact moment of the start of the repayment period, the borrower can skip paying the last coupon amount and keep the last payment instead.

The protocol allows the creation of a coupon bullet loan, toward which the borrower repays a part of the loan interest in regular intervals before finally repaying the principal amount. To account for late repayment, the protocol defines a repayment period duration over which the borrower can repay the loan before it reaches maturity. This is shown in figure 5.1.

Upon calling the repay function to repay the entire loan, the function will calculate and attempt to repay any outstanding coupon payments before paying the leftover principal amount, as shown in figure 5.2.

The full repayment will first calculate how many coupon payments the borrower has left and pay them.

However, the contracts are missing a check for the case where the repayment timestamp is equal to the start of the repayment period. In this case, the calculation will return one less coupon payment but still allow for principal repayment.

This will allow the borrower to repay the principal, which will transition the loan to the REPAID state, allowing the borrower to keep the last coupon payment for themselves.

**Exploit Scenario**

Alice has a coupon bullet loan with a target origination amount of 1,000,000 USDC that is borrowed at a 10% interest rate and lasts for 10 days. She needs to pay 10,000 USDC in coupon payments, and then pay the last 1,000,000 USDC as principal.

Alice makes the first eight coupon payments and sets up a script to repay the principal at the exact timestamp when the repayment period starts.

Her script runs successfully, skipping the last coupon payment and repaying the principal amount. Alice keeps the last 1,000 USDC payment for herself.

**Recommendations**

Short term, update the check in LoanLogic to account for the case where the repayment timestamp is equal to the start of the repayment period:

referenceTimestamp >= loanMaturity - repaymentPeriodDuration

Long term, explicitly specify preconditions and postconditions for all functions to more easily identify what is being checked and what needs to be checked in a function. Set up fuzzing tests with Echidna to identify unexpected behavior.

## Lenders’ unborrowed deposits can be locked up by a borrower

**Description**

Due to an incorrect check in the TickLogic library contract, any lender whose position is borrowed for less than the minimum deposit amount will have their entire deposit locked up. The lenders’ deposits will be unlocked if the loan is repaid or their position becomes borrowed for an amount greater than the minimum deposit amount.

The protocol allows lenders to freely withdraw the unborrowed part of their deposit during an active loan by using the detach function. Before a withdrawal is made, the function checks that the position value at the end of the loan, subtracted by the unborrowed amount to be withdrawn, is greater than or equal to the minimum deposit amount

However, this check will prevent the withdrawal of unborrowed assets for any lenders whose position is borrowed for less than the minimum deposit amount. This allows the borrower to either intentionally or accidentally lock up the lenders’ funds.

**Exploit Scenario**

Assume a Revolving Credit Line is created for Alice with a minimum deposit amount of 1 ether and a loan duration of one year.

1. Ten lenders deposit 10 ether each into the pool at a 5% rate before the first borrow is made.

2. Alice borrows 9 ether. Since all of the lenders have deposited at the same rate and the same epoch, their positions are equally borrowed by 0.9 ether.

Although each lender has 9.1 ether in unborrowed assets, they are prevented from detaching their unborrowed assets due to their position being borrowed for less than the minimum deposit amount. This allows Alice to force the lenders into keeping their funds in the pool.

**Recommendations**

Short term, revise the check to allow lenders to withdraw their unborrowed amount as long as the sum of the leftover amount and the borrowed amount left in the contract is larger than or equal to the minimum deposit amount.

Long term, use extensive smart contract fuzzing to test that users are never blocked from performing expected operations.

## optOut can be called multiple times

**Description**

Lenders are allowed to trigger an opt out operation multiple times over the same loan, with unclear implications.

When a lender wishes not to be a part of the next loan cycle, they can call the optOut function to signal their intent. This function will validate that the position has been borrowed and that the loan has not passed maturity.

To compute the borrowedAmount, the getPositionRepartition function is called. However, this function does not validate that the lender has not yet opted out of their position if the loan has not yet been repaid

This allows a malicious lender to arbitrarily inflate the optedOut variable, which could break internal accounting

**Exploit Scenario**

Eve, a malicious lender, calls the optOut function multiple times. This inflates the adjustedOptOut amount variable of her position drastically. Bob, a borrower, is unable to repay his loan due to an underflow when calculating the tickAdjusted variable in the prepareTickForNextLoan function.

**Recommendations**

Short term, ensure that users cannot call the optOut function again if they have already opted out of the next loan cycle.

Long term, improve unit test coverage and implement smart contract fuzzing to uncover potential edge cases and ensure intended behavior throughout the system.

## Risks related to deflationary, inflationary, or rebasing tokens

**Description**

The loan product contract and the pool custodian contract do not check that the expected amount has been sent on transfer. This can lead to incorrect accounting in both contracts if they do not receive enough assets because tokens are inflationary, deflationary, or rebase.

Each lending product uses a pool custodian contract that holds custody of all ERC20 assets and through which the product contract deposits and withdraws the assets

safeTransferFrom does not check that the expected value was transferred. If the tokens take a fee, the amount transferred can be less than expected. Similarly, the product contracts assume that the received amount is the same as the inputted amount. As a result, the custodian contract may hold fewer tokens than expected, and the accounting of the product may be incorrect.

**Exploit Scenario**

USDT enables a 0.5% fee on transfers. Every deposit to a product leads to the creation of positions with more tokens than received. As a result, the positions become insolvent.

**Recommendations**

Short term, check the custodian balance during deposits and withdraws, and adjust the expected position amounts. Be aware that some commonly used assets, such as USDT, can enable fees in the future. Additionally, prevent the usage of rebasing tokens.

Long term, review the token integration checklist in Appendix I, and ensure that the system is robust against edge cases to the ERC20 specification.

## Rounding down when computing fees benefits users

**Description**

Fees are computed using an incorrect rounding direction that reduces the amount collected by the protocol. As a result, fees may be less than intended when being transferred to the protocol.

After the borrow function has finished its primary execution, it calculates the total dust value in the contracts, then attributes allocation to pools.

To compute the fees, the registerBorrowingFees function is called.

However, this function does not seem to be doing any division; in fact, it uses the mul function from FixedPointMathLib, which is implemented using mulDivDown with the denominator as 10 ** token.decimals(). The division will not be exact most of the time, reducing the computed value.

**Exploit Scenario**

Alice deploys a pool, which requires a certain fee. Over time, a large number of users interact with the pool, but the rounding down of the fee computation produces less fees than expected.

**Recommendations**

Short term, ensure that the fees computation rounds in favor of the protocol.

Long term, improve unit test coverage and implement smart contract fuzzing to uncover potential edge cases and ensure intended behavior throughout the system.

## Lenders can prevent each other from earning interest

**Description**

A lender can prevent another lender's position from earning interest by opting out of their position and then updating their position's rate after the loan is repaid.

A lender can prevent their position from being borrowed in any future lending cycles by opting out of their position. An opted-out position will not be borrowed and can be withdrawn only once the loan is repaid; however, the protocol also allows the lender to update the rate of their position even though the position can never be borrowed.

This occurs because the updateRate function lacks a check that the lender has opted out of their position, as shown in figures 11.1 and 11.2.

Updating the rate of an opted-out position will internally withdraw the amount held in the position using the current rate and then deposit it using a new rate.

After depositing using the new rate, the system will consider the amount held in the opted-out position as borrowable; however, the getPositionCurrentValue function will still return the entire amount as unborrowed. This allows the lender to withdraw their position.

This means that if a lender’s position is borrowed, they can withdraw another lender’s assets. The other lender will still be able to withdraw their assets if the loan is repaid or if there are more unborrowed assets in the pool, but they will not earn interest.

**Exploit Scenario**

A Revolving Credit Loan is created for Alice with a maximum borrow amount of 50 ether with a loan duration of 10 days.
1. Bob deposits 50 ether into the pool, which Alice borrows.
2. Bob opts out of his position, causing the system to consider his position unborrowable for future loans.
3. Charlie sees that there are no other lenders in the pool with borrowable assets and deposits 50 ether into the pool.
4. Alice repays her loan.
5. Bob updates the rate of his position to be lower than Charlie’s.
6. Alice borrows another 50 ether, starting the next loan cycle.
7. Since Bob’s position had a lower rate, his position is borrowed first. Bob then withdraws the 50 ether because the system incorrectly considers his position as not borrowed.
8. There is no ether left in the pool, so Charlie is not able to withdraw his assets, and his position will not earn interest for the duration of the loan.

**Recommendations**

Short term, update the validation in updateRate to prevent lenders who have opted out of their positions to update the rates of the positions.

Long term, improve unit test coverage and come up with a list of system properties that can be tested with smart contract fuzzing. This issue could be discovered by implementing a property test that checks that an opted-out position can never be borrowed from again.

## Detached positions are incorrectly calculated

**Description**

Lenders who detach their position earn less interest in subsequent loan cycles due to an incorrect calculation in getAdjustedAmount.

The protocol allows users whose position has not been fully borrowed to detach their position and withdraw their leftover unborrowed assets. To avoid storing the state of each individual position, the protocol derives the position’s borrowed and unborrowed amounts. This is done via the adjustedAmount and the total available and borrowed amount of that interest rate, as shown below:

The getAdjustedAmount function has a special case for calculating detached positions using the position information and the endOfLoanYieldFactor of the loan in which the position was detached

However, this calculation returns an incorrect value, causing the protocol to consider the position’s borrowed amount to be less than it actually is. This will result in the lender earning less interest than they should.

**Exploit Scenario**

A Revolving Credit Line is created for Alice with a maximum borrowable amount of 10 ether and a duration of 10 weeks.

1. Bob deposits 15 ether at a 10% rate.
2. Alice borrows 10 ether from the pool.
3. Bob detaches his position, withdrawing the unborrowed 5 ether to his account.
4. Alice repays the loan on time and takes another loan of 10 ether.
5. Although Alice took a 10 ether loan, the protocol considers Bob’s position to be borrowed for 9.9998 ether.
6. Bob earns less accruals than he should for the duration of the loan and all subsequent loans.

**Recommendations**

Short term, investigate the ways in which detached position values are calculated and fees are subtracted in the system to ensure position values are derived correctly.

Long term, consider redesigning how fees are charged throughout the system so that derived values are not incorrectly impacted. Improve unit test coverage and come up with a list of system properties that can be tested with smart contract fuzzing. For example, this issue could have been discovered by implementing a property test that checks that the total borrowed amount should be equal to the sum of the borrowed amounts of all positions.

## Borrower can reduce lender accruals

**Description**

A borrower can reduce the accrued interest of lenders by borrowing a smaller amount of assets multiple times.

The protocol allows borrowers to borrow any amount of assets up to the maximum borrowable amount defined for that product instance. Once the assets have been borrowed, the protocol uses the tick yield factor for each interest rate to calculate the accrued interest of lender positions. Whenever a borrow action occurs using a certain interest rate, the protocol calculates the new yield factor increase and adds it to the previous yield factor of that interest rate:

However, since the yieldFactor has the same precision as the token and the function uses fixed-point division that rounds down, the yield factor increase will truncate and return a lower-than-expected value for lower-precision tokens (e.g., USDC). This precision loss is compounded during multiple borrow actions and leads to loss of accruals for lenders.

**Exploit Scenario**

A Revolving Credit Loan is created for Alice with a maximum borrowable amount of 10 million USDC and a duration of 10 weeks. Bob deposits 10 million USDC at a 10% rate.

If Alice were to borrow the entire amount at once and repay on time, at the end of the loan she would need to repay ~10,191,780 USDC and Bob would earn 10,191,780 USDC

If Alice were to borrow 100,000 USDC one hundred times and repay on time, at the end of the loan she would have to repay ~10,191,780 USDC, but Bob would have earned only 10,191,000 USDC, or 780 fewer USDC than if the entire amount was borrowed at once.

Assuming a gas cost of 100 gwei and a block gas limit of 30 million gas, Alice can perform ~500 smaller borrow actions within the block gas limit, which would cost her approximately 3 MATIC or around 3.4 USD. If Alice filled two blocks with ~1,000 borrows total, it would reduce Bob's accruals by 1,780 USDC but would cost Alice only ~6.5 MATIC or ~7.3 USDC.

**Recommendations**

Short term, consider increasing the precision of the yieldFactor and evaluate which rounding direction should be used to minimize the impact of the precision loss.

Long term, analyze the system arithmetic for precision issues and decide on the rounding direction of each operation so that precision loss is minimized. Improve unit test coverage and thoroughly test the system using assets with different decimal precision. Use smart contract fuzzing to test system invariants. This issue could have been discovered by implementing a property test; with a total borrowed amount X and a duration Y, the total accruals should be equal to Z, no matter if the amount was borrowed at once or via multiple smaller amounts.

## Borrower can start a lending cycle before deposits are made

**Description**

A borrower can initiate a loan before any deposits are made by borrowing 10 or fewer tokens.

The borrow function of the RCLBorrower implements a check for precision issues in an effort to avoid disabling the function due to compounding rounding errors.

However, this also allows the borrower to initiate a loan without any deposits being made. This is because the remainingAmount equals the amount passed into the function, which allows the borrow function to succeed even though the borrowedAmount is zero.

**Recommendations**

Short term, implement a check that ensures the borrowAmount is always greater than zero.

Long term, investigate and minimize any rounding issues present in the protocol. Explicitly specify preconditions and postconditions of all functions to more easily identify what is being checked and what needs to be checked in a function. Set up fuzzing tests with Echidna to identify unexpected behavior.

## Missing validation in detach

**Description**

The detach function lacks validation that the position being detached has been partially borrowed.

The detach function is intended to allow lenders to withdraw their unborrowed assets from a partially borrowed position. The function fetches the unborrowed and borrowed amounts of the position and performs validation on them:

The execution will continue until it reaches the highlighted line in figure 17.2, at which point it will revert due to a division-by-zero error, since the endOfLoanValue is equal to zero:

Although the function validates that a position is borrowed, it may not properly validate all cases. If a lender attempts to detach a completely unborrowed position, the function will revert before reaching the validation on line 547 of figure 17.1.

**Recommendations**

Short term, add validation to the detach function that reverts with a descriptive error if the position being detached has not been borrowed.

Long term, explicitly specify preconditions and postconditions of all functions to more easily identify what is being checked and what needs to be checked in a function. Set up fuzzing tests with Echidna to identify unexpected behavior. This issue could have been discovered by implementing a unit test that checks that detaching an unborrowed position reverts with the proper error message.

## Roles manager can never be updated

**Description**

The Atlendis Labs protocol uses role-based access control to manage permissions and privileges. The manager of the roles is responsible for granting and revoking roles; if necessary, the role manager can be transferred to another address by governance via the updateRolesManager function. However, the updateRolesManager function will always revert:

**Exploit Scenario**

The PoolTokenFeesController contract is deployed with an incorrect rolesManager. Governance notices the issue and attempts to call updateRolesManager with the correct _rolesManager address. However, the call will revert because the if statement in the onlyGovernance modifier and the if statement in updateRolesManager are contradictory.

**Recommendations**

Short term, remove the if statement, as the condition it checks is incorrect, and use just the onlyGovernance modifier instead.

Long term, improve unit test coverage and create a list of system properties that can be tested with smart contract fuzzing.

## Problematic approach to the handling precision errors

**Description**

Many functions in the TickLogic contract contain obscure precision checks. These checks may protect against a subset of unexpected behavior, but are problematic when considering the full range of inputs.

One such example is in the RCLBorrower contract’s borrow function. After iterating over the ticks and updating respective amounts, the RCLBorrower contract checks the liquidity remaining after adjustment. If the amount is less than or equal to 10, the execution of the borrow function continues; otherwise, it halts. This effectively allows users to borrow an additional 10 units of the tokens:

When a user withdraws, similar logic applies to the withdrawal amounts, as defined in Figure 24.2. This allows users to withdraw up to 10 tokens for free.

This check is also present in the exit function of the TickLogic contract, which gives users up to 10 free tokens upon exiting their position:

This attack can be significantly cheaper on layer two networks, where gas costs allow attackers to submit many transactions and profit from a series of deposits or withdrawals.

**Recommendations**

Short term, avoid comparing against magic values and always ensure that any deposits into the protocol round up to maximize the amount a user pays, and that any withdrawals or exits round down to minimize the amount a user receives. Solving precision errors through rounding will ensure that dust, or small amounts of imprecision, does not allow attackers to profit from interacting with the system.

Long term, analyze all formulas in the system to ensure that they follow accurate rounding directions, as explained in Appendix C: Rounding Recommendations.

## Lenders with larger deposits earn less accruals if their position is only partially borrowed

**Description**

If a lender deposits more assets than the amount that ends up being borrowed from their position, they will receive fewer accruals than if they had deposited exactly the amount that ends up being borrowed.

The protocol allows lenders to deposit any amount of assets at any allowed interest rate, as long as the amount is larger than the minimum deposit amount. It uses the interest rate yield factor to calculate the amount of interest/accruals the lender has earned during the lending cycle. Whenever a borrow is triggered, a yield factor increase is calculated and added to the previous yieldFactor:

However, the larger the amount in tick.baseEpochsAmounts.adjustedDeposits, the less the yield factor will increase due to rounding errors. This will lead to lenders earning less interest if their deposit is significantly larger than the borrower amount.

**Exploit Scenario**

An RCL is created for Alice with a maximum borrowable amount of 30 million USDC and a duration of 10 weeks. 

Consider the following two cases:

1. Bob deposits 10 million USDC, and his position is fully borrowed. At the end of the loan, Bob has earned 191,780 USDC.
2. Bob deposits 30 million USDC, and his position is borrowed for 10 million USDC. At the end of the loan, Bob has earned 191,760 USDC.

Bob has earned 20 USDC less in case two than in case one.

**Recommendations**

Short term, investigate the cause of the rounding error and mitigate it. Thoroughly test the interest calculations using tokens with different decimal precisions and in various different scenarios.

Long term, analyze the system arithmetic for precision issues and decide on the rounding direction of each operation to minimize precision loss. Use smart contract fuzzing to test system invariants. This issue could have been discovered by implementing a property test that checks that the accrued interest of a lender is equal any time their position is borrowed for a certain amount, no matter the lender’s deposited amount.

## Lack of zero-value checks on functions

**Description**

Certain setter functions fail to validate incoming arguments; therefore, callers of these functions can accidentally set important state variables to the zero address.

For example, the constructor function in the Hyper contract, which sets the WETH contract, lacks zero-value checks.

If the WETH address is set to the zero address, the admin must redeploy the Hyper contracts to reset the address’s value.

**Exploit Scenario**

Alice deploys a new version of the Hyper contract but mistakenly enters the zero address for the WETH address. She must redeploy the system to reset the value of WETH.

**Recommendations**

Short term, add zero-value checks to all function arguments to ensure that users cannot accidentally set incorrect values, misconfiguring the system.

Long term, use the Slither static analyzer to catch common issues such as this one. Consider integrating a Slither scan into the project’s continuous integration pipeline, pre-commit hooks, or build scripts.

## Risk of token theft due to possible integer underflow in slt

**Description**

An attacker can steal funds from a pool using a function that fails to revert on unexpected input.

The addSignedDelta function is used by the allocate, unallocate, and unstake functions to alter the liquidity of a pool. These functions lack checks to ensure that the input values are within permissible limits; instead, they rely on the addSignedDelta function to revert on unexpected inputs.

The addSignedDelta function checks for the sign of the delta value; it subtracts its two’s complement from the input value if delta is a negative integer. After the subtraction operation, the code checks for underflow using the slt (signed-less-than) function from the EVM dialect of the YUL language. The slt function assumes that both arguments are in a two’s complement representation of the integer values and checks their signs using the most significant bit. If an underflow occurs in the previous subtraction operation, slt’s output value will be interpreted as a negative integer with its most significant bit set to 1. Because the slt function will find that the output value is less than the input value based on their signs, it will return 1. However, the addSignedDelta function expects the result of slt to be 0 when an underflow occurs; therefore, it fails to capture the underflow condition correctly.

**Exploit Scenario**

Alice allocates 100 USDC into a pool. Eve, an attacker, calls the unallocate function with the 100 USDC as the deltaLiquidity argument. The addSignedDelta function fails to revert on the integer underflow that results, and Eve is able to withdraw Alice’s assets.

**Recommendations**

Short term, take one of the following actions:

● Correct the implementation of the addSignedDelta function to account for underflows.
● Use high-level Solidity code instead of assembly code to avoid further issues; assembly code does not support sub 256-bit types.

Regardless of which action is taken, add test cases to verify the correctness of the new implementation; add both unit test cases and fuzz test cases to capture all of the edge cases.

Long term, carefully review the codebase to find assembly code and verify the correctness of these assembly code blocks by adding test cases. Do not rely on certain behavior of assembly code while working with sub 256-bit types because this behavior is not defined and can change at any time.

Note: Do not consider using the lt function in place of the slt function because it is not sufficient to capture all of the overflow and underflow conditions.

## Risk of token theft due to unchecked type conversion

**Description**

An attacker can steal funds from a pool using a function that fails to revert on unexpected input.

The addSignedDelta function is used by the allocate, unallocate, and unstake functions to alter the liquidity of a pool. These functions lack checks to ensure that the input values are within permissible limits; instead, they rely on the addSignedDelta function to revert on unexpected inputs.

When the value of delta is a positive integer, the result of the add function (from the EVM dialect of the YUL language) will overflow; however, the code cannot capture this overflow. This is because the arguments of the function are 128-bit types, but the assembly code does not have types and operates on 256-bit values. The add function returns a 256-bit value as its result. The addition of two 128-bit integers can never overflow a 256-bit integer. For this reason, the result of the add function will never wrap around the maximum value of a 256-bit integer, and the slt function will never find the output value to be less than the input value, which means it will never return 1 to indicate that an overflow has occurred. As a result, the code fails to capture the overflow condition correctly.

There are multiple ways in which an attacker could use this issue to withdraw more liquidity than they have deposited in a pool.

**Exploit Scenario1**

Alice and Bob allocate 500 USDC each into a USDC-ETH pool. The total allocated liquidity is 1,000 USDC. Eve, an attacker, unallocates 1,000 USDC from the entire pool, withdrawing everyone’s assets.

**Exploit Scenario 2**

Alice allocates 100 USDC into a pool. Eve calls the unstake function to increase the value of her own liquidity without depositing any assets and then calls unallocate to withdraw the funds from the pool.

**Recommendations**

Short term, take one of the following actions:

● Correct the implementation of the addSignedDelta function to account for overflows.
● Use high-level Solidity code instead of assembly code to avoid further issues; assembly code does not support sub 256-bit types.

Regardless of which action is taken, add test cases to verify the correctness of the new implementation; add both unit test cases and fuzz test cases to capture all of the edge cases.

Long term, carefully review the codebase to find assembly code and verify the correctness of these assembly code blocks by adding test cases. Do not rely on certain behavior of assembly code while working with sub 256-bit types because this behavior is not defined and can change at any time.

## Users can swap without paying any fees

**Description**

While the swap function is being processed, the input and output values of the swap are calculated with and without the fee amount assessed from the total. However, if the user’s requested amount exceeds the maxInput amount, the re-addition of the swap fee to deltaInput is missed, allowing the user to swap without paying the fee.

The swap function calls the _swapExactIn function, which contains logic to save intermediate values while processing token swaps. The deltaInput variable is initially used to derive the nextIndependent value without the fee amount assessed. That fee amount should be added back to deltaInput afterward so that it can be added to the total amount withdrawn from the user later in the process. The fee amount is added back to deltaInput when _swap.remainder is less than or equal to maxInput, but not if _swap.remainder is greater than maxInput. If the fee is not added, then when deltaInput is added to _swap.input, the fee will not be represented in that amount, which means that it will not be withdrawn from the user.

**Exploit Scenario**

Eve executes a swap in which the remainder is greater than the maximum input. Due to the calculations in the swap function, Eve can swap without paying the fee amount.

**Recommendations**

Short term, fix the function to factor in feeAmounts when the value of remainder is greater than the value of maxInput.

Long term, thoroughly analyze the system to identify invariants related to proper fee assessment. Fuzz those invariants using Echidna to ensure that the functions return the expected values and that they are accurate.

## Swap function returns incorrectly scaled output token amount

**Description**

The swap function’s output value is given per unit of liquidity in the given pool, but it is not scaled by the pool’s total liquidity. As a result, users will not receive the number of tokens that they expect on swaps

The output token value is calculated using the difference between the liveDependent and nextDependent variables, both of which are calculated using the reserve amount of the input token. However, the output value is not multiplied by the total liquidity value of the pool, so the output amount is scaled incorrectly

This causes the number of output tokens to be either too few or too many, depending on the current amount of liquidity in the pool.

Primitive also discovered this issue during the code review.

**Exploit Scenario**

Alice swaps WETH for USDC using Hyper. The pool has less than 1 wad of liquidity. The token output value that is returned to Alice is less than what it should be.

**Recommendations**

Short term, revise the swap function so that it multiplies the output token by the total liquidity present in the pool in which the swap takes place.

Long term, identify additional system invariants and fuzz them using Echidna to ensure that the functions return the expected values and that they are accurate.

## Pool strike price could be zero due to lack of lower bound check on maxTick

**Description**

The maxTick variable is used to approximate the strike price of a pool. However, the code does not validate the lower bound of maxTick, which means that the strike price of a pool can be 0, causing the pool’s assets to be mispriced.

The maxTick variable is provided by pool creators and is validated on pool creation by the checkParameters function, which checks that the volatility, maxTick, duration, jit, and priorityFee values are within safe bounds. However, for the maxTick parameter, the validation function checks only its upper bound:

The maxTick value is used to calculate prices, including the strike price of an asset

However, because the lower bound of maxTick is not checked, the computePriceWithTick function could return a 0 value for the price, which would cause the system to use the incorrect value for strike prices.

**Exploit Scenario**

Alice creates a pool with a maxTick value of -887272. Upon calculating the strike price at maturity, the tickWad and price values are calculated as follows:

**Recommendations**

Short term, bound maxTick to a lower bound that will not allow strike prices to converge to 0, and have strike prices round up to ensure that they can never be 0.

Long term, clearly document the expected and unexpected flows in the contract to ensure that users are aware of expected behavior.

## Rounding error allows liquidity to be added without depositing tokens

**Description**

In pools that have an asset token of six decimals, small allocations of those tokens will be scaled by the number of decimals and then rounded down. A deltaAsset value of 0 can be returned by the getLiquidityDeltas function, even if a nonzero deltaLiquidity argument is provided, allowing an attacker to add liquidity without transferring any tokens.

In the calculation of amountAssetDec in the getAmounts function, the amount of the asset token in wad units (1e18) is scaled to a value representative of that token’s decimals and then rounded down. If amountAssetWad is a small value, amountAssetDec is rounded down to 0 and returned, tricking the system into thinking zero asset tokens are required to fulfill liquidity allocation.

**Exploit Scenario**

Eve, an attacker, finds or creates a pool where the asset token has six decimals. She calls the allocate function and adds the smallest possible unit (1) as the amount to that pool. The getAmounts function scales the value to six decimals, rounds down, and returns 0 for amountAssetDec, which is then multiplied by deltaLiquidity; as a result, 0 is returned for the required deltaAsset. The parameters of the pool are changed and the _increaseReserves function is called with the correct deltaQuote value but 0 for the deltaAsset value.

**Recommendations**

Short term, make one of the following changes:

● Have amounts of token allocations rounded up to the nearest decimal unit depending on the token’s assigned decimal value (e.g., for a token with six decimals, amounts should round up to the next token decimal point, which would be 1e12 = 1e(18 - 6)).
● Add a zero-value check on the return values of getAmounts.

Be careful to consider the downstream implications of any short-term fixes implemented for this issue, as the getAmounts function is used in critical system operations. 

Long term, continue to add unit tests that consider the expected outcomes of a wide array of inputs and scenarios. Document all of the assumptions within the system and implement fuzz testing for them with Echidna in order to catch edge cases like this that might break assumptions that are not readily apparent.

## Limited precision in strike prices due to fixed tick spacing

**Description**

The strike price is allowed to take on values only from a predefined set of values. This is a fixed pricing grid with limited precision, which means that the strike price can deviate from a desired price.

The strike price in Hyper is supplied through an int24 tick value that maps to a price in the pricing grid, which was computed from the tick base value using the exponential function (price = TICK_BASE ^ tick). As a result, the price values are spaced out exponentially from each other.

The risk of price deviation is evident when looking at the resulting values from one tick to another. Given a price in the range of 30,000 per quote token (e.g., BTC/USD), the price difference from one tick to another is approximately 3 USD. The difference between the ticks grows exponentially by one part per thousand.

Further imprecisions could compound when computing the next tick from the price after a swap. However, this is not an exploitable issue, as the next tick is not actually used to derive the next price in the protocol.

**Exploit Scenario**

Alice opens a pool with a USD/BTC pair and wants to set a price of 30,003 USD/BTC. Due to the limited precision in the ticks, the strike price ends up being set to 30,000 USD/BTC.

**Recommendations**

Short term, document whether this is desired behavior and describe the limitations and rounding issues that could result from it.

Long term, consider whether a fixed price grid is necessary; if it is not, consider using the decimals representation for storing prices.

## getAmountOut returns incorrect value when called by controller

**Description**

When a controller contract calls the getAmountOut function, the feeAmount should be calculated using the priorityFee value; however, the function incorrectly uses the fee value. This causes the function to return an incorrect output value.

When a controller contract swaps tokens, the fee assessed in the transaction is calculated using the priorityFee parameter.

The purpose of the getAmountOut function is to return the expected token amount that the user would receive after executing a swap. Using the wrong feeAmount will skew the output calculation, causing a discrepancy between what the user expects to receive and what they actually receive after executing the swap.

**Exploit Scenario**

Alice wants to swap two tokens from a pool that has a controller contract set. She queries the controller with the proposed swap parameters, and the controller contract calls the getAmountOut function with those same parameters. The getAmountOut function returns an output amount of five tokens. Alice sends a swap transaction to the controller, which calls the swap function on the Hyper contract. Only four tokens are returned to Alice instead of the expected five.

**Recommendations**

Short term, refactor the getAmountOut function to accurately mirror calculations made in the swap function and to use priorityFee when a controller calls getAmountOut.

Long term, expand the current unit test suite to ensure that data returned by view functions is accurate and up to date.

## Mismatched base unit comparison can inflate limit tolerance

**Description**

When a user swaps tokens, the code enforces the user’s limitPrice value, denominated in the token’s decimal units, by comparing it to the value of the pool’s lastPrice value, denominated in wad units. The discrepancy between these units could prevent a user’s intended price limit from being enforced, resulting in a swap at a market rate that the user did not intend to swap at.

When a user calls the swap function, the limit argument is used in the _swapExactIn function as the limitPrice value (figure 20.1); the limitPrice value determines whether the user’s intended limit price has been exceeded. The user’s assumption is that this value is denominated in the quote token’s decimal units. The limitPrice value is compared to the nextPrice value, taken from the pool’s lastPrice value, and if the user’s price limit has been met or exceeded, the swap reverts.

The lastPrice variable is denominated in wad units, so it has 18 decimals of precision. If the quote token used to denominate limitPrice has any fewer than 18 decimals, then it will always be smaller than intended compared to nextPrice. As a result, a swap transaction submitted by a user who has met or exceeded their price limit will not revert as expected.

**Exploit Scenario**

Alice swaps tokens in a pool in which both tokens have six decimals. She sets a price limit of seven A tokens for one B token. The trade causes the pricing formula to swing to nine A tokens for one B token. The _swapExactIn function checks whether 7e6 is greater than 9e18, which it is not. No SwapLimitReached error is thrown as a result, and the swap is allowed to complete despite the surpassed price limit.

**Recommendations**

Short term, modify the associated code so that either the limitPrice input value is scaled to wad units or the lastPrice value is scaled to however many decimals the quote token has. Clearly document for users the denomination they should use for the price limit.

Long term, expand the current unit test suite to consider token pools with all ranges of decimals; for each scenario, ensure that the swap function will revert when a price limit is reached.

## Lack of zero-value check on constructors and initializers

**Description**

Several contracts’ constructors and initialization functions fail to validate incoming arguments. As a result, important state variables could be set to the zero address, which would result in the loss of assets.

These constructors include that of the SmartVaultManager contract, which sets the _masterWallet address (figure 3.1). SmartVaultManager contract is the entry point of the system and is used by users to deposit their tokens. User deposits are transferred to the _masterWallet address (figure 3.2).

If _masterWallet is set to the zero address, the tokens will be transferred to the zero address and will be lost permanently.

The constructors and initialization functions of the following contracts also fail to validate incoming arguments:

● StrategyRegistry
● DepositSwap
● SmartVault
● SmartVaultFactory
● SpoolAccessControllable
● DepositManager
● RiskManager
● SmartVaultManager
● WithdrawalManager
● RewardManager
● RewardPool
● Strategy

**Exploit Scenario**

Bob deploys the Spool system. During deployment, Bob accidentally sets the _masterWallet parameter of the SmartVaultManager contract to the zero address. Alice, excited about the new protocol, deposits 1 million WETH into it. Her deposited WETH tokens are transferred to the zero address, and Alice loses 1 million WETH.

**Recommendations**

Short term, add zero-value checks on all constructor arguments to ensure that the deployer cannot accidentally set incorrect values.

Long term, use Slither , which will catch functions that do not have zero-value checks.

## Upgradeable contracts set state variables in the constructor

**Description**

The state variables set in the constructor of the RewardManager implementation contract are not visible in the proxy contract, making the RewardManager contract unusable. The same issue exists in the RewardPool and Strategy smart contracts.

Upgradeable smart contracts using the delegatecall proxy pattern should implement an initializer function to set state variables in the proxy contract storage. The constructor function can be used to set immutable variables in the implementation contract because these variables do not consume storage slots and their values are inlined in the deployed code.

The RewardManager contract is deployed as an upgradeable smart contract, but it sets the state variable _assetGroupRegistry in the constructor function.

The value of the _assetGroupRegistry variable will not be visible in the proxy contract, and the admin will not be able to add reward tokens to smart vaults, making the RewardManager contract unusable.

The following smart contracts are also affected by the same issue:

1. The ReentrancyGuard contract, which is non-upgradeable and is extended by RewardManager
2. The RewardPool contract, which sets the state variable allowUpdates in the constructor
3. The Strategy contract, which sets the state variable StrategyName in the constructor

**Exploit Scenario**

Bob creates a smart vault and wants to add a reward token to it. He calls the addToken function on the RewardManager contract, but the transaction unexpectedly reverts.

**Recommendations**

Short term, make the following changes:

1. Make _assetGroupRegistry an immutable variable in the RewardManager contract.
2. Extend the ReentrancyGuardUpgradeable contract in the RewardManager contract.
3. Make allowUpdates an immutable variable in the RewardPool contract.
4. Move the statement _strategyName = strategyName_; from the Strategy contract’s constructor to the contract’s __Strategy_init function.
5. Review all of the upgradeable contracts to ensure that they extend only upgradeable library contracts and that the inherited contracts have a __gap storage variable to prevent storage collision issues with future upgrades.

Long term, review all of the upgradeable contracts to ensure that they use the initializer function instead of the constructor function to set state variables. Use slither-check-upgradeability to find issues related to upgradeable smart contracts.

## Insu cient validation of oracle price data

**Description**

The current validation of the values returned by Chainlink’s latestRoundData function could result in the use of stale data.

The latestRoundData function returns the following values: the answer , the roundId (which represents the current round), the answeredInRound value (which corresponds to the round in which the answer was computed), and the updatedAt value (which is the timestamp of when the round was updated). An updatedAt value of 0 means that the round is not complete and should not be used. An answeredInRound value that is less than the roundId indicates stale data.

However, the _getAssetPriceInUsd function in the UsdPriceFeeManager contract does not check for these conditions.

**Exploit Scenario**

The price of an asset changes, but the Chainlink price feed is not updated correctly. The system uses the stale price data, and as a result, the asset is not correctly distributed in the strategies.

**Recommendations**

Short term, have _getAssetPriceInUsd perform the following sanity check: require(updatedAt != 0 && answeredInRound == roundId) . This check will ensure that the round has finished and that the pricing data is from the current round.

Long term, when integrating with third-party protocols, make sure to accurately read their documentation and implement the appropriate sanity checks.

## Incorrect handling of fromVaultsOnly in removeStrategy

**Description**

The removeStrategy function allows Spool admins to remove a strategy from the smart vaults using it. Admins are also able to remove the strategy from the StrategyRegistry contract, but only if the value of fromVaultsOnly is false ; however, the implementation enforces the opposite, as shown in figure 6.1

**Exploit Scenario**

Bob, a Spool admin, calls removeStrategy with fromVaultsOnly set to true , believing that this call will not remove the strategy from the StrategyRegistry contract. However, once the transaction is executed, he discovers that the strategy was indeed removed.

**Recommendations**

Short term, replace if (fromVaultsOnly) with if (!fromVaultsOnly) in the removeStrategy function to implement the expected behavior.

Long term, improve the system’s unit and integration tests to catch issues such as this one.

## Risk of LinearAllocationProvider and ExponentialAllocationProvider reverts due to division by zero

**Description**

The LinearAllocationProvider and ExponentialAllocationProvider contracts’ calculateAllocation function can revert due to a division-by-zero error: LinearAllocationProvider ’s function reverts when the sum of the strategies’ APY values is 0 , and ExponentialAllocationProvider ’s function reverts when a single strategy has an APY value of 0 .

Figure 7.1 shows a snippet of the LinearAllocationProvider contract’s calculateAllocation function; if the apySum variable, which is the sum of all the strategies’ APY values, is 0 , a division-by-zero error will occur.

Figure 7.2 shows that for the ExponentialAllocationProvider contract’s calculateAllocation function, if the call to log_2 occurs with partApy set to 0 , the function will revert because of log_2 ’s require statement shown in figure 7.3.

**Exploit Scenario**

Bob deploys a smart vault with two strategies using the ExponentialAllocationProvider contract. At some point, one of the strategies has 0 APY, causing the transaction call to reallocate the assets to unexpectedly revert.

**Recommendations**

Short term, modify both versions of the calculateAllocation function so that they correctly handle cases in which a strategy’s APY is 0 .

Long term, improve the system’s unit and integration tests to ensure that the basic operations work as expected.

## Risk of malformed calldata of calls to guard contracts

**Description**

The GuardManager contract does not pad custom values while constructing the calldata for calls to guard contracts. The calldata could be malformed, causing the affected guard contract to give incorrect results or to always revert calls.

Guards for vaults are customizable checks that are executed on every user action. The result of a guard contract either approves or disapproves user actions. The GuardManager contract handles the logic to call guard contracts and to check their results (figure 10.1).

The arguments of the runGuards function include information related to the given user action and custom values defined at the time of guard definition. The GuardManager.setGuards function initializes the guards in the GuardManager contract.

Using the guard definition, the GuardManager contract manually constructs the calldata with the selected values from the user action information and the custom values (figure 10.2).

However, the contract concatenates the custom values without considering their lengths and required padding. If these custom values are not properly padded at the time of guard initialization, the call will receive malformed data. As a result, either of the following could happen:

1. Every call to the guard contract will always fail, and user action transactions will always revert. The smart vault using the guard will become unusable.
2. The guard contract will receive incorrect arguments and return incorrect results. Invalid user actions could be approved, and valid user actions could be rejected.

**Exploit Scenario**

Bob deploys a smart vault and creates a guard for it. The guard contract takes only one custom value as an argument. Bob created the guard definition in GuardManager without padding the custom value. Alice tries to deposit into the smart vault, and the guard contract is called for her action. The call to the guard contract fails, and the transaction reverts. The smart vault is unusable.

**Recommendations**

Short term, modify the associated code so that it verifies that custom values are properly padded before guard definitions are initialized in GuardManager.setGuards . 

Long term, avoid implementing low-level manipulations. If such implementations are unavoidable, carefully review the Solidity documentation before implementing them to ensure that they are implemented correctly. Additionally, improve the user documentation with necessary technical details to properly use the system.

## Lack of contract existence checks on low-level calls

**Description**

The GuardManager and Swapper contracts use low-level calls without contract existence checks. If the target address is incorrect or the contract at that address is destroyed, a low-level call will still return success.

The Swapper.swap function uses the address().call(...) function to swap tokens (figure 13.1).

Therefore, if the swapTarget address is incorrect or the target contract has been destroyed, the execution will not revert even if the swap is not successful.

We rated this finding as only a low-severity issue because the Swapper contract transfers the unswapped tokens to the receiver if a swap is not successful. However, the CompoundV2Strategy contract uses the Swapper contract to exchange COMP tokens for underlying tokens (figure 13.3).

If the swap operation fails, the COMP will stay in CompoundV2Strategy . This will cause users to lose the yield they would have gotten from compounding. Because the swap operation fails silently, the “do hard worker” may not notice that yield is not compounding. As a result, users will receive less in profit than they otherwise would have. 

The GuardManager.runGuards function, which uses the address().staticcall() function, is also affected by this issue. However, the return value of the call is decoded, so the calls would not fail silently.

**Exploit Scenario**

The Spool team deploys CompoundV2Strategy with a market that gives COMP tokens to its users. While executing the doHardWork function for smart vaults using CompoundV2Strategy , the “do hard worker” sets the swapTarget address to an incorrect address. The swap operation to exchange COMP to the underlying token fails silently. The gained yield is not deposited into the market. The users receive less in profit.

**Recommendations**

Short term, implement a contract existence check before the low-level calls in GuardManager.runGuards and Swapper.swap .

Long term, avoid implementing low-level calls. If such calls are unavoidable, carefully review the Solidity documentation , particularly the “Warnings” section, before implementing them to ensure that they are implemented correctly.

## Transfers of D-NFTs result in double counting of SVT balance

**Description**

The _activeUserNFTIds and _activeUserNFTCount variables are not updated for the sender account on the transfer of NFTs. As a result, SVTs for transferred NFTs will be counted twice, causing the system to show an incorrect SVT balance.

The _afterTokenTransfer hook in the SmartVault contract is executed after every token transfer to update information about users’ active NFTs:

When a user transfers an NFT to another user, the function adds the NFT ID to the active NFT IDs of the receiver’s account but does not remove the ID from the active NFT IDs of the sender’s account. Additionally, the active NFT count is not updated for the sender’s account.

The getUserSVTBalance function of the SmartVaultManager contract uses the SmartVault contract’s _activeUserNFTIds array to calculate a given user’s SVT balance:

Because transferred NFT IDs are active for both senders and receivers, the SVTs corresponding to the NFT IDs will be counted for both users. This double counting will keep increasing the SVT balance for users with every transfer, causing an incorrect balance to be shown to users and third-party integrators.

**Exploit Scenario**

Alice deposits assets into a smart vault and receives a D-NFT. Alice's assets are deposited into the protocols after doHardWork is called. Alice transfers the D-NFT to herself. The SmartVault contract adds the D-NFT ID to _activeUserNFTIds for Alice again. Alice checks her SVT balance and sees double the balance she had before.

**Recommendations**

Short term, modify the _afterTokenTransfer function so that it removes NFT IDs from the active NFT IDs for the sender’s account when users transfer D-NFTs and W-NFTs.

Long term, add unit test cases for all possible user interactions to catch issues such as this.

## Flawed loop for syncing flushes results in higher management fees

**Description**

The loop used to sync flush indexes in the SmartVaultManager contract computes an inflated value of the oldTotalSVTs variable, which results in higher management fees paid to the smart vault owner.

The _syncSmartVault function in the SmartVaultManager contract implements a loop to process every flush index from flushIndex.toSync to flushIndex.current

This loop adds the value of mintedSVTs to the newSVTs variables and then computes the value of oldTotalSVTs by adding newSVTs to it in every iteration. Because mintedSVTs are added in every iteration, new minted SVTs are added for each flush index multiple times when the loop is iterated more than once.

The value of oldTotalSVTs is then passed to the syncDeposit function of the DepositManager contract, which uses it to compute the management fee for the smart vault. The use of the inflated value of oldTotalSVTs causes higher management fees to be paid to the smart vault owner.

**Exploit Scenario**

Alice deposits assets into a smart vault and flushes it. Before doHardWork is executed, Bob deposits assets into the same smart vault and flushes it. At this point, flushIndex.current has been increased twice for the smart vault. After the execution of doHardWork , the loop to sync the smart vault is iterated twice. As a result, a double management fee is paid to the smart vault owner, and Alice and Bob lose assets.

**Recommendations**

Short term, modify the loop so that syncResult.mintedSVTs is added to bag.oldTotalSVTs instead of bag.newSVTs . 

Long term, be careful when implementing accumulators in loops. Add test cases for multiple interactions to catch such issues.

## Incorrect ghost strategy check

**Description**

The emergencyWithdraw and redeemStrategyShares functions incorrectly check whether a strategy is a ghost strategy after checking that the strategy has a ROLE_STRATEGY role.

A ghost strategy will never have the ROLE_STRATEGY role, so both functions will always incorrectly revert if a ghost strategy is passed in the strategies array.

**Exploit Scenario**

Bob calls redeemStrategyShares with the ghost strategy in strategies and the transaction unexpectedly reverts.

**Recommendations**

Short term, modify the affected functions so that they verify whether the given strategy is a ghost strategy before checking the role with _checkRole .

Long term, clearly document which roles a contract should have and implement the appropriate checks to verify them.

## Reallocation process reverts when a ghost strategy is present

**Description**

The reallocation process reverts in multiple places when a ghost strategy is present. As a result, it is impossible to reallocate a smart vault with a ghost strategy.

The first revert would occur in the mapStrategies function (figure 26.1). Users calling the reallocate function would not know to add the ghost strategy address in the strategies array, which holds the strategies that need to be reallocated. This function reverts if it does not find a strategy in the array. Even if the ghost strategy address is in strategies , a revert would occur in the areas described below.

During the reallocation process, the doReallocation function calls the beforeRedeemalCheck and beforeDepositCheck functions even on ghost strategies (figure 26.2); however, their implementation is to revert on ghost strategies with an IsGhostStrategy error (figure 26.3).

**Exploit Scenario**

A strategy is removed from a smart vault. Bob, who has the ROLE_ALLOCATOR role, calls reallocate , but it reverts and the smart vault is impossible to reallocate.

**Recommendations**

Short term, modify the associated code so that ghost strategies are not passed to the reallocate function in the _smartVaultStrategies parameter.

Long term, improve the system’s unit and integration tests to test for smart vaults with ghost strategies. Such tests are currently missing.

## Reward emission can be extended for a removed reward token

**Description**

Smart vault owners can extend the reward emission for a removed token, which may cause tokens to be stuck in the RewardManager contract. The removeReward function in the RewardManager contract calls the _removeReward function, which does not remove the reward configuration:

The extendRewardEmission function checks whether the value of tokenAdded in the rewardConfiguration[smartVault][token] configuration is not zero to make sure that the token was already added to the smart vault:

After removing a reward token from a smart vault, the value of tokenAdded in the rewardConfiguration[smartVault][token] configuration is left as nonzero, which allows the smart vault owner to extend the reward emission for the removed token.

**Exploit Scenario**

Alice adds a reward token A to her smart vault S. After a month, she removes token A from her smart vault. After some time, she forgets that she removed token A from her vault. She calls extendRewardEmission with 1,000 token A as the reward. The amount of token A is transferred from Alice to the RewardManager contract, but it is not distributed to the users because it is not present in the list of reward tokens added for smart vault S. The 1,000 tokens are stuck in the RewardManager contract.

**Recommendations**

Short term, modify the associated code so that it deletes the rewardConfiguration[smartVault][token] configuration when removing a reward token for a smart vault.

Long term, add test cases to check for expected user interactions to catch bugs such as this.

## A reward token cannot be added once it is removed from a smart vault

**Description**

Smart vault owners cannot add reward tokens again after they have been removed once from the smart vault, making owners incapable of providing incentives to users. 

The removeReward function in the RewardManager contract calls the _removeReward function, which does not remove the reward configuration:

The addToken function checks whether the value of tokenAdded in the rewardConfiguration[smartVault][token] configuration is zero to make sure that the token was not already added to the smart vault:

After a reward token is removed from a smart vault, the value of tokenAdded in the rewardConfiguration[smartVault][token] configuration is left as nonzero, which prevents the smart vault owner from adding the token again for reward distribution as an incentive to the users of the smart vault.

**Exploit Scenario**

Alice adds a reward token A to her smart vault S. After a month, she removes token A from her smart vault. Noticing the success of her earlier reward incentive program, she wants to add reward token A to her smart vault again, but her transaction to add the reward token reverts, leaving her with no choice but to distribute another token.

**Recommendations**

Short term, modify the associated code so that it deletes the rewardConfiguration[smartVault][token] configuration when removing a reward token for a smart vault.

Long term, add test cases to check for expected user interactions to catch bugs such as this.

## ExponentialAllocationProvider reverts on strategies without risk scores

**Description**

The ExponentialAllocationProvider.calculateAllocation function can revert due to division-by-zero error when a strategy’s risk score has not been set by the risk provider.

The risk variable in calculateAllocation represents the risk score set by the risk provider for the given strategy, represented by the index i . Ghost strategies can be passed to the function. If a ghost strategy’s risk score has not been set (which is likely, as there would be no reason to set one), the function will revert with a division-by-zero error.

**Exploit Scenario**

A strategy is removed from a smart vault that uses the ExponentialAllocationProvider contract. Bob, who has the ROLE_ALLOCATOR role, calls reallocate ; however, it reverts, and the smart vault is impossible to reallocate.

**Recommendations**

Short term, modify the calculateAllocation function so that it properly handles strategies with uninitialized risk scores.

Long term, improve the unit and integration tests for the allocators. Refactor the codebase so that ghost strategies are not passed to the calculateAllocator function.

## Removing a strategy makes the smart vault unusable

**Description**

Removing a strategy from a smart vault causes every subsequent deposit transaction to revert, making the smart vault unusable.

The deposit function of the SmartVaultManager contract calls the depositAssets function on the DepositManager contract. The depositAssets function calls the checkDepositRatio function, which takes an argument called strategyRatios

The value of strategyRatios is fetched from the StrategyRegistry contract, which returns an empty array on ghost strategies. This empty array is then used in a for loop in the calculateFlushFactors function:

The statement calculating the value of normalization tries to access an index of the empty array and reverts with the Index out of bounds error, causing the deposit function to revert for every transaction thereafter.

**Exploit Scenario**

A Spool admin removes a strategy from a smart vault. Because of the presence of a ghost strategy, users’ deposit transactions into the smart vault revert with the Index out of bounds error.

**Recommendations**

Short term, modify the calculateFlushFactors function so that it skips ghost strategies in the loop used to calculate the value of normalization.

Long term, review the entire codebase, check the effects of removing strategies from smart vaults, and ensure that all of the functionality works for smart vaults with one or more ghost strategies.

## Risk of DoS due to unbounded loops

**Description**

Guards and actions are run in unbounded loops. A smart vault creator can add too many guards and actions, potentially trapping the deposit and withdrawal functionality due to a lack of gas.

The runGuards function calls all the configured guard contracts in a loop:

Multiple conditions can cause this loop to run out of gas:

 - ● The vault creator adds too many guards.
 ● One of the guard
   contracts consumes a high amount of gas.
   ● A guard starts consuming a
   high amount of gas after a specific block or at a specific state.

If user transactions reach out-of-gas errors due to these conditions, smart vaults can become unusable, and funds can become stuck in the protocol.

A similar issue affects the runActions function in the AuctionManager contract.

**Exploit Scenario**

Eve creates a smart vault with an upgradeable guard contract. Later, when users have made large deposits, Eve upgrades the guard contract to consume all of the available gas to trap user deposits in the smart vault for as long as she wants.

**Recommendations**

Short term, model all of the system's variable-length loops, including the ones used by runGuards and runActions , to ensure they cannot block contract execution within expected system parameters.

Long term, carefully audit operations that consume a large amount of gas, especially those in loops.

## Unsafe casts throughout the codebase

**Description**

The codebase contains unsafe casts that could cause mathematical errors if they are reachable in certain states. Examples of possible unsafe casts are shown in figures 38.1 and 38.2.

**Recommendations**

Short term, review the codebase to identify all of the casts that may be unsafe. Analyze whether these casts could be a problem in the current codebase and, if they are unsafe, make the necessary changes to make them safe.

Long term, when implementing potentially unsafe casts, always include comments to explain why those casts are safe in the context of the codebase.


## Incorrect inflator arithmetic in view functions

**Description**

The return values of the loansInfo method in the Pool contract include a maxThresholdPrice that has already been multiplied by the value of inflator.

The maxThresholdPrice value returned by this function is used in two places to calculate the highest threshold price (HTP): the htp() function (figure 3.2) and the poolPricesInfo() function (figure 3.3). However, in both cases the value is incorrectly multiplied by the value of inflator a second time , causing the value of htp to be overstated.

The htp value is used, among other things, to determine whether a new loan can be drawn. It is compared against the value of lup , which cannot be lower than the value of htp . If these htp numbers are used to determine whether a new loan can be entered, the code may consider that loan invalid when it is actually valid.

**Exploit Scenario**

Alice wants to initiate a new loan. The UI uses the return value of PoolInfoUtils.htp() to determine whether the loan is valid. The new lup value from Alice's loan is above the actual htp value but below the incorrect htp value that resulted from the double multiplication of maxThresholdPrice . As a result, the UI prevents her from submitting the loan, and Alice has to use a competitor to get her loan.

**Recommendations**

Short term, update the htp() and poolPricesInfo() functions in PoolInfoUtils so that they do not multiply maxThresholdPrice by the value of inflator twice.

Long term, add tests to ensure that all functions return correct values.

## Missing checks of array lengths in LP allowance update functions

**Description**

The increaseLPAllowance and decreaseLPAllowance functions both accept array arguments, indexes_ and amounts_

There is no check to ensure the array arguments are equal in length. This means that the functions would accept two arrays of different lengths. If the amounts_ array is longer than indexes_ , the extra amounts_ values will be ignored silently.

**Exploit Scenario**

Alice wants to reduce allowances to Eve for various buckets, including a large bucket she is particularly concerned about. Alice inadvertently omits the large bucket index from indexes_ but does include the amount in her call to decreaseLPAllowance() . The transaction completes successfully but does not impact the allowance for the large bucket. Eve drains the large bucket.

**Recommendations**

Short term, add a check to both increaseLPAllowance and decreaseLPAllowance to ensure that the lengths of amounts_ and indexes_ are the same.

Long term, carefully consider the mistakes that could be made by users to help design data validation strategies for external functions.

## Issues with Chainlink oracle’s return data validation

**Description**

Chainlink oracles are used to compute the price of a collateral token throughout the protocol. When validating the oracle's return data, the returned price is compared to the price of the previous round.

However, there are a few issues with the validation:

 - ● The increase of the currentRoundId value may not be statically
   increasing across rounds. The only requirement is that the roundID
   increases monotonically.
   ● The updatedAt value in the oracle response
   is never checked, so potentially stale data could be coming from the
   priceAggregator contract.
   ● The roundId and answeredInRound values in
   the oracle response are not checked for equality, which could
   indicate that the answer returned by the oracle is fresh.

**Exploit Scenario**

The Chainlink oracle attempts to compare the current returned price to the price in the previous roundID . However, because the roundID did not increase by one from the previous round to the current round, the request fails, and the price oracle returns a failure. A stale price is then used by the protocol.

**Recommendations**

Short term, have the code validate that the timestamp value is greater than 0 to ensure that the data is fresh. Also, have the code check that the roundID and answeredInRound values are equal to ensure that the returned answer is not stale. Lastly check that the timestamp value is not decreasing from round to round.

Long term, carefully investigate oracle integrations for potential footguns in order to conform to correct API usage.

## Inconsistent use of safeTransfer for collateralToken

**Description**

The Raft contracts rely on ERC-20 tokens as collateral that must be deposited in order to mint R tokens. However, although the SafeERC20 library is used for collateral token transfers, there are a few places where the safeTransfer function is missing:

 - ● The transfer of collateralToken in the liquidate function in the
   PositionManager contract:
   ● The transfer of stETH in the
   managePositionStETH function in the PositionManagerStETH contract:

**Exploit Scenario**

Governance approves an ERC-20 token that returns a Boolean on failure to be used as collateral. However, since the return values of this ERC-20 token are not checked, Alice, a liquidator, does not receive any collateral for performing a liquidation.

**Recommendations**

Short term, use the SafeERC20 library’s safeTransfer function for the collateralToken .

Long term, improve unit test coverage to uncover edge cases and ensure intended behavior throughout the protocol.

## Price deviations between stETH and ETH may cause Tellor oracle to return an incorrect price

**Description**

Finding ID: TOB-RAFT-6 The Raft finance contracts rely on oracles to compute the price of the collateral tokens used throughout the codebase. If the Chainlink oracle is down, the Tellor oracle is used as a backup. However, the Tellor oracle does not use the stETH/USD price feed. Instead it uses the ETH/USD price feed to determine the price of stETH. This could be problematic if stETH depegs, which can occur during black swan events.

**Exploit Scenario**

Alice has a position in the system. A significant black swan event causes the depeg of staked Ether. As a result, the Tellor oracle returns an incorrect price, which prevents Alice's position from being liquidated despite being eligible for liquidation.

**Recommendations**

Short term, carefully monitor the Tellor oracle, especially during any sort of market volatility.

Long term, investigate the robustness of the oracles and document possible circumstances that could cause them to return incorrect prices.

## Liquidation rewards are calculated incorrectly


**Description**

Whenever a position's collateralization ratio falls between 100% and 110%, the position becomes eligible for liquidation. A liquidator can pay off the position's total debt to restore solvency. In exchange, the liquidator receives a liquidation reward for removing bad debt, in addition to the amount of debt the liquidator has paid off. However, the calculation performed in the split function is incorrect and does not reward the liquidator with the matchingCollateral amount of tokens:

**Exploit Scenario**

Alice, a liquidator, attempts to liquidate an insolvent position. However, upon liquidation, she receives only the liquidationReward amount of tokens, without the matchingCollateral . As a result her liquidation is unprofitable and she has lost funds.

**Recommendations**

Short term, have the code compute the collateralToSendToLiquidator variable as liquidationReward + matchingCollateral.

Long term, improve unit test coverage to uncover edge cases and ensure intended behavior throughout the protocol.

## Replay protection missing in castVoteWithReasonAndParamsBySig

**Description**

The castVoteWithReasonAndParamsBySig function does not include a voter nonce, so transactions involving the function can be replayed by anyone.

Votes can be cast through signatures by encoding the vote counts in the params argument.

The castVoteWithReasonAndParamsBySig function calls the _countVoteFractional function in the GovernorCountingFractional contract, which keeps track of partial votes. Unlike _countVoteNominal , _countVoteFractional can be called multiple times, as long as the voter’s total voting weight is not exceeded.

**Exploit Scenario**

Alice has 100,000 voting power. She signs a message, and a relayer calls castVoteWithReasonAndParamsBySig to vote for one “yes” and one “abstain”. Eve sees this transaction on-chain and replays it for the remainder of Alice’s voting power, casting votes that Alice did not intend to.

**Recommendations**

Short term, either include a voter nonce for replay protection or modify the _countVoteFractional function to require that _proposalVotersWeightCast[proposalId][account] equals 0 , which would allow votes to be cast only once.

Long term, increase the test coverage to include cases of signature replay.

## Ability to lock any user’s tokens using deposit_for

**Description**

The deposit_for function can be used to lock anyone's tokens given sufficient token approvals and an existing lock.

The same issue is present in the veCRV contract for the CRV token, so it may be known or intentional.

**Exploit Scenario**

Alice gives unlimited FXS token approval to the veFXS contract. Alice wants to lock 1 FXS for 4 years. Bob sees that Alice has 100,000 FXS and locks all of the tokens for her. Alice is no longer able to access her 100,000 FXS.

**Recommendations**

Short term, make users aware of the issue in the existing token contract. Only present the user with exact approval limits when locking FXS.

## receiveFlashLoan does not account for fees

**Description**

The receiveFlashLoan functions of the scWETHv2 and scUSDCv2 vaults ignore the Balancer flash loan fees and repay exactly the amount that was loaned. This is not currently an issue because the Balancer vault does not charge any fees for flash loans. However, if Balancer implements fees for flash loans in the future, the Sandclock vaults would be prevented from withdrawing investments back into the vault.

In the Balancer flashLoan function , shown in figure 1.1, the contract calls the recipient’s receiveFlashLoan function with four arguments: the addresses of the tokens loaned, the amounts for each token, the fees to be paid for the loan for each token, and the calldata provided by the caller.

The Sandclock vaults ignore the fee amount and repay only the principal, which would lead to reverts if the fees are ever changed to nonzero values. Although this problem is present in multiple vaults, the receiveFlashLoan implementation of the scWETHv2 contract is shown in figure 1.2 as an illustrative example:

**Exploit Scenario**

After Sandclock’s scUSDv2 and scWETHv2 vaults are deployed and users start depositing assets, the Balancer governance system decides to start charging fees for flash loans. Users of the Sandclock protocol now discover that, apart from the float margin, most of their funds are locked because it is impossible to use the flash loan functions to withdraw vault assets from the underlying investment pools.

**Recommendations**

Short term, use the feeAmounts parameter in the calculation for repayment to account for future Balancer flash loan fees. This will prevent unexpected reverts in the flash loan handler function.

Long term, document and justify all ignored arguments provided by external callers. This will facilitate a review of the system’s third-party interactions and help prevent similar issues from being introduced in the future.

## Reward token distribution rate can diverge from reward token balance

**Description**

The privileged distributor role is responsible for transferring reward tokens to the RewardTracker contract and then passing the number of tokens sent as the _reward parameter to the notifyRewardAmount method. However, the _reward parameter provided to this method can be larger than the number of reward tokens transferred. Given the accounting for leftover rewards, such a situation would be difficult to recover from.

If a _reward value smaller than the actual number of transferred tokens is provided, the situation can be fixed by calling notifyRewardAmount again with a _reward parameter that accounts for the difference between the RewardTracker contract’s actual token balance and the rewards already scheduled for distribution. This solution is possible because the _notifyRewardAmount helper function accounts for leftover rewards if it is called during an ongoing reward period.

This accounting for leftover rewards, however, makes the situation difficult to recover from if a _reward parameter that is too large is provided to the notifyRewardAmount method. As shown by the arithmetic in figure 2.2, if the reward period has not finished, the code for creating the newRewardRate value can only add to the reward distribution, not subtract from it. The only way to bring a too-large reward distribution back in line with the RewardTracker contract’s reward token balance is to transfer additional reward tokens to the contract.

**Exploit Scenario**

The RewardTracker distributor transfers 10 reward tokens to the RewardTracker contract and then mistakenly calls the notifyRewardAmount method with a _reward parameter of 100. Some users call the claimRewards method early and receive inflated rewards until the contract’s balance is depleted, leaving later users unable to claim any rewards. To recover, the distributor either needs to provide another 90 reward tokens to the RewardTracker contract or accept the reputational loss of allowing this misconfigured reward period to finish before resetting the reward payouts correctly during the next period.

**Recommendations**

Short term, modify the _notifyRewardAmount helper function to reset the rewardRate so that it is in line with the current rewardToken balance and the time remaining in the reward period. This change could also allow the fetchRewards method to maintain its current behavior but with only a single rewardToken.balanceOf external call.

Long term, review the internal accounting state variables and document the ways in which they are influenced by the actual flow of funds. Pay attention to any internal accounting values that can be influenced by external sources, including privileged accounts, and reexamine the system’s assumptions surrounding the flow of funds.

## Miscalculation in beforeWithdraw can leave the vault with less than minimum float

**Description**

When a user wants to redeem or withdraw, the beforeWithdraw function is called with the number of assets to be withdrawn as the assets parameter. This function makes sure that if the value of the float parameter (that is, the available assets in the vault) is not enough to pay for the withdrawal, the strategy gets some assets back from the pools to be able to pay.

When the float value is enough, the function returns and the withdrawal is paid with the existing float. If the float value is not enough, the missing amount is recovered from the pools via the adapters.

The issue lies in the calculation of the missing parameter: it does not guarantee that the float value remaining after the withdrawal is at least the value of the minimumFloatAmount parameter. The consequence is that the calculation always leaves a f loat equal to floatRequired in the vault. If this value is small enough, it can cause users to waste gas when withdrawing small amounts because they will need to pay for the gas-intensive _withdrawToVault action. This eclipses the usefulness of having the float in the vault.

The correct calculation should be uint256 missing = assets + minimumFloat - float; . Using this correct calculation would make the calculation of the floatRequired parameter unnecessary as it would no longer be required or used in the rest of the code.

**Exploit Scenario**

The value for minimumFloatAmount is set to 1 ether in the scWETHv2 contract. For this scenario, suppose that the current float is exactly equal to minimumFloatAmount .

Alice wants to withdraw 0.15 WETH from her invested amount. Because this amount is less than the current float, her withdrawal is paid from the vault assets, leaving the float equal to 0.85 WETH after the operation.

Then, Bill wants to withdraw 0.9 WETH, but the vault has no available assets to pay for it. In this case, when beforeWithdraw is called, Bill has to pay gas for the call to _withdrawToVault , which is an expensive action because it includes gas-intensive operations such as loops and a flash loan.

After Bill’s withdrawal, the float in the vault is 0.15 WETH. This is a relatively small amount compared with minimumFloatValue , and it will likely make the next withdrawing/redeeming user also have to pay for the call to _withdrawToVault

**Recommendations**

Short term, replace the calculation of the missing amount to be withdrawn on line 393 of the scWETHv2 contract with assets + minimumFloat - float . This calculation will ensure that the minimum float restriction is enforced after withdrawals. It will take the required f loat into consideration, so the separate calculation of floatRequired on line 392 of scWETHv2 would no longer be required.

Long term, add unit or fuzz tests to make sure that the vault has an amount of assets equal to or greater than the minimum expected amount at all times.

## Last user in scWETHv2 vault will not be able to withdraw their funds

**Description**

When a user wants to withdraw, the withdrawal amount is checked against the current vault float (the uninvested assets readily available in the vault). If the withdrawal amount is less than the float, the amount is paid from the available balance; otherwise, the protocol has to disinvest from the strategies to get the required assets to pay for the withdrawal.

The issue with this approach is that in order to maintain a float equal to the minimumFloatValue parameter in the vault, the value to be disinvested from the strategies is calculated in the beforeWithdraw function, and its correct value is equal to the sum of the amount to be withdrawn and the minimum float minus the current float. If there is only one user remaining in the vault and they want to withdraw, this enforcement will not allow them to do so, because there will not be enough invested in the strategies to leave a minimum float in the vault after the withdrawal. They would only be able to withdraw their assets minus the minimum float at most.

The code for the _withdrawToVault function is shown in figure 4.1. The line highlighted in the figure would cause the revert in this situation, as there would not be enough invested to supply the requested amount.

Additionally, when this revert occurs, an integer overflow is given as the reason, which obscures the real reason and can make the user’s experience more confusing.

**Exploit Scenario**

Bob is the only remaining user in a scWETHv2 vault, and he has 2 ether invested. He wants to withdraw his assets, but all of his calls to the withdrawal function keep reverting due to an integer overflow.

He keeps trying, wasting gas in the process, until he discovers that the maximum amount he is allowed to withdraw is around 1 ether. The rest of his funds are locked in the vault until the keeper makes a manual call to withdrawToVault or until the admin lowers the minimum float value.

**Recommendations**

Short term, fix the calculation of the amount to be withdrawn and make sure that it never exceeds the total invested amount.

Long term, add end-to-end unit or fuzz tests that are representative of the way multiple users can interact with the protocol. Test for edge cases involving various numbers of users, investment amounts, and critical interactions, and make sure that the protocol’s invariants hold and that users do not lose access to funds in the event of such edge cases.

## Lido stake rate limit could lead to unexpected reverts

**Description**

To mitigate the effects of a surge in demand for stETH on the deposit queue, Lido has implemented a rate limit for stake submissions. This rate limit is ignored by the lidoSwapWethToWstEth method of the Swapper library, potentially leading to unexpected reversions.

The Lido stETH integration guide states the following:

 - To avoid [reverts due to the rate limit being hit], you should check
   if getCurrentStakeLimit() >= amountToStake , and if it's not you can
   go with an alternative route.

**Exploit Scenario**

A surge in demand for Ethereum validators leads many people using Lido to stake ETH, causing the Lido rate limit to be hit, and the submit method of the stEth contract begins to revert. As a result, the Sandclock keeper is unable to deposit despite the presence of alternate routes to obtain stETH, such as through Curve or Balancer.

**Recommendations**

Short term, have the lidoSwapWethToWstEth method of the Swapper library check whether the amount being deposited is less than the value returned by the getCurrentStakeLimit method of the stEth contract. If it is not, have the code use ZeroEx to swap or revert with a message that clearly communicates the reason for the failure.

Long term, review the documentation for all third-party interactions and note any situations in which the integration could revert unexpectedly. If such reversions are acceptable, clearly document how they could occur and include a justification for this acceptance in the inline comments.

## Chainlink oracles could return stale price data

**Description**

The latestRoundData() function from Chainlink oracles returns five values: roundId , answer , startedAt , updatedAt , and answeredInRound . The PriceConverter contract reads only the answer value and discards the rest. This can cause outdated prices to be used for token conversions, such as the ETH-to-USDC conversion shown in figure 6.1.

According to the Chainlink documentation , if the latestRoundData() function is used, the updatedAt value should be checked to ensure that the returned value is recent enough for the application.

Similarly, the LUSD/ETH price feed used by the scLiquity vault is an intermediate contract that calls the deprecated latestAnswer method on upstream Chainlink oracles.

The Chainlink API reference flags the latestAnswer method as “(Deprecated - Do not use this function.).” Note that the upstream IPriceFeed contracts called by the intermediate LSUDUsdToLUSDEth contract are upgradeable proxies. It is possible that the implementations will be updated to remove support for the deprecated latestAnswer method, breaking the scLiquity vault’s lusd2eth price feed.

Because the oracle price feeds are used for calculating the slippage tolerance, a difference may exist between the oracle price and the DEX pool spot price, either due to price update delays or normal price fluctuations or because the feed has become stale. This could lead to two possible adverse scenarios:

 - ● If the oracle price is significantly higher than the pool price,
   the slippage tolerance could be too loose, introducing the
   possibility of an MEV sandwich attack that can profit on the excess.
   ● If the oracle price is significantly lower than the pool price, the
   slippage tolerance could be too tight, and the transaction will
   always revert. Users will perceive this as a denial of service
   because they would not be able to interact with the protocol until
   the price difference is settled.

**Exploit Scenario**

Bob has assets invested in a scWETHv2 vault and wants to withdraw part of his assets. He interacts with the contracts, and every withdrawal transaction he submits reverts due to a large difference between the oracle and pool prices, leading to failed slippage checks. This results in a waste of gas and leaves Bob confused, as there is no clear indication of where the problem lies.

**Recommendations**

Short term, make sure that the oracles report up-to-date data, and replace the external LUSD/ETH oracle with one that supports verification of the latest update timestamp. In the case of stale oracle data, pause price-dependent Sandclock functionality until the oracle comes back online or the admin replaces it with a live oracle.

Long term, review the documentation for Chainlink and other oracle integrations to ensure that all of the security requirements are met to avoid potential issues, and add tests that take these possible situations into account.

## Lack of lower and upper bounds for system parameters

**Description**

The lack of lower and upper bound checks when setting important system parameters could lead to a temporary denial of service, allow users to complete their withdrawals prematurely, or otherwise hinder the expected performance of the system.

The setWithdrawalDelay function of the RootERC20PredicateFlowRate contract can be used by the rate control role to set the amount of time that a user needs to wait before they can withdraw their assets from the root chain of the bridge.

The withdrawalDelay variable value is applied to all currently pending withdrawals in the system, as shown in the highlighted lines of figure 2.2.

However, the setWithdrawalDelay function does not contain any validation on the delay input parameter. If the input parameter is set to zero, users can skip the withdrawal queue and immediately withdraw their assets. Conversely, if this variable is set to a very high value, it could prevent users from withdrawing their assets for as long as this variable is not updated.

The setRateControlThreshold allows the rate control role to set important token parameters that are used to limit the amount of tokens that can be withdrawn at once, or in a certain time period, in order to mitigate the risk of a large amount of tokens being bridged after an exploit.

However, because the _setFlowRateThreshold function of the FlowRateDetection contract is missing upper bounds on the input parameters, these values could be set to an incorrect or very high value. This could potentially allow users to withdraw large amounts of tokens at once, without triggering the withdrawal queue.

**Exploit Scenario**

Alice attempts to update the withdrawalDelay state variable from 24 to 48 hours. However, she mistakenly sets the variable to 0 . Eve uses this setting to skip the withdrawal queue and immediately withdraws her assets.

**Recommendations**

Short term, determine reasonable lower and upper bounds for the setWithdrawalDelay and setRateControlThreshold functions, and add the necessary validation to those functions.

Long term, carefully document which system parameters are configurable and ensure they have adequate upper and lower bound checks.

## RootERC20Predicate is incompatible with nonstandard ERC-20 tokens

**Description**

The deposit and depositTo functions of the RootERC20Predicate contract are incompatible with nonstandard ERC-20 tokens, such as tokens that take a fee on transfer.

The RootERC20Predicate contract allows users to deposit arbitrary tokens into the root chain of the bridge and mint the corresponding token on the child chain of the bridge. Users can deposit their tokens by approving the bridge for the required amount and then calling the deposit or depositTo function of the contract. These functions will call the internal _depositERC20 function, which will perform a check to ensure the token balance of the contract is exactly equal to the balance of the contract before the deposit, plus the amount of tokens that are being deposited.

However, some nonstandard ERC-20 tokens will take a percentage of the transferred amount as a fee. Due to this, the require statement highlighted in figure 3.1 will always fail, preventing users from depositing such tokens.

**Recommendations**

Short term, clearly document that nonstandard ERC-20 tokens are not supported by the protocol. If the team determines that they want to support nonstandard ERC-20 implementations, additional logic should be added into the _deposit function to determine the actual token amount received by the contract. In this case, reentrancy protection may be needed to mitigate the risks of ERC-777 and similar tokens that implement callbacks whenever tokens are sent or received.

Long term, be aware of the idiosyncrasies of ERC-20 implementations. This standard has a history of misuses and issues.

## Net asset value of fractional and leverage token may reflect invalid price

**Description**

The contracts for the fToken and xToken minted by the Treasury expose a method, nav , to retrieve the net asset value (NAV) of tokens. This represents what portion of Treasury’s reserve collateral is available to be redeemed in exchange for the given token. However, the NAV uses currentBaseTokenPrice indiscriminately and fails to validate that the price is valid. As outlined in the aforementioned issue ( TOB-ADFX-4 ), fetching the TWAP using the Action.None value may return an invalid price. External integrations that rely on this price may misreport the true NAV or use the incorrect value in calculations, and will therefore get incorrect results.

**Recommendations**

Short term, consider reverting if the price is deemed invalid rather than reflecting the invalid price in the net asset value, or add an additional return value that external integrations can use to determine whether the NAV is based on an invalid price.

Long term, perform consistent validation throughout the codebase and add tests for scenarios where the price is considered invalid.

## Sum of all user shares does not equal total supply

**Description**

The FxInitialFund contract allows users to deposit up until the mint occurs, then funds can be withdrawn. Because users cannot deposit after withdrawals are enabled, the totalSupply is never updated and is therefore not accurate after withdrawals.

While this does not affect the pro-rata share of fToken and xToken that each user receives, it does make the total supply of shares inaccurate. Rather than recompute the proportion of tokens a share is entitled to each time and hold the total supply constant, the amount of fToken and xToken each share is worth can be cached in the mint function, and the total supply can be decremented each time a withdrawal is made. This accomplishes the same functionality for withdrawals and ensures that the total supply of shares is accurately tracked.

**Recommendations**

Short term, compute the amount of fToken and xToken to distribute per share in the mint function, and update the withdraw function to use this amount and decrement the total supply of shares.

Long term, ensure that user balances and the sum of all user balances, total supply, is synchronized, and implement invariant testing. This can also be applied to other balances, such as rewards per individual and total available rewards.

## Lack of validation when updating system configurations

**Description**

The market contract’s Boolean configurations do not validate that the configuration has changed when they are updated. While setting the same value is benign, it may obscure a logical error in a peripheral program that would be readily identified if the update reverts and raises an alarm.

While the stability ratio cannot be too large, the _updateStabilityRatio function does not validate that the stability ratio is greater than 100% (1e18), which would allow the protocol to be under-collateralized.

**Recommendations**

Short term, require that the new value is not equal to the old and that the stability ratio is not less than 100%.

Long term, validate that system parameters are within sane bounds and that updates are not no-ops.

## Lack of slippage checks prevents user from specifying acceptable loss

**Description**

The router contract to interact with the FxUSD contract and Market contract does not allow specifying slippage limits for redeeming xToken and mint FxUSD, respectively. Note, there is a slippage check for the target asset in the Facet contract but not for the intermediary assets exchanged during the multi-leg “swap”.

This also affects the Balancer wrapper contract, which was not in scope of our review.

**Recommendations**

Short term, allow users to specify a slippage tolerance for all paths.

Long term, do not hard-code arguments that users should be able to input, especially ones that protect against losses.

## Deployments to L2 should check sequencer uptime for Chainlink price feeds

**Description**

The protocol’s current usage of Chainlink price feeds does not consider whether the sequencer is down for deployments to layer 2 blockchains (L2s) such as Arbitrum and Optimism. This can result in the usage of stale price info and unfairly impact users. For example, it may be appropriate to allot a grace period for xToken holders before allowing them to redeem their proportion of the base collateral. However, insolvent deeply underwater treasuries should likely still be liquidated to avoid further losses.

**Exploit Scenario**

Holders of xToken are attempting to re-collateralize the system to prevent their NAV from becoming $0, but the L2’s sequencer is down. An outdated price is used and fToken holders begin redeeming their portion of the treasury’s collateral without sufficient time for xToken holders to respond once the sequencer is up.

**Recommendations**

Short term, implement functionality to check the sequencer uptime with Chainlink oracles for deploying the f(x) protocol to L2s.

Long term, validate that oracle prices are sufficiently fresh and not manipulated.

## Validation of system invariants is error prone

**Description**

The architecture of the f(x) protocol is interleaved and its interactions are complex. The interdependencies of the components are not managed well, and validations are distributed in disparate components rather than maintaining logical separation. We recommend simplifying the architecture and refactoring assertions about how the state is updated to be closely tied to where the state is updated. This is especially important in light of the concerns related to stability and incentives and lack of specification, as logical errors and economic issues may be coupled together. Making these investments will strengthen the security posture of the system and facilitate invariant testing.

The Treasury does not validate the postcondition that the collateral ratio increases when minting xToken and redeeming fToken. In stability mode, liquidations perform fToken redemptions to attempt re-collateralizing the protocol to a collateral ratio above the stability ratio.

The Treasury does not validate the postcondition that the collateral ratio is not below stability ratio, allowing immediate liquidations when minting fToken. The credit health is softly enforced by the maxMintableFToken math library function used in the Market and by the FxUSD contract (when enabled), but it would be more robust to strictly require the collateral ratio has not fallen too far when performing mint actions to prevent insolvency. This applies to redeeming xToken as well since it is also expected to reduce the collateral ratio like minting fToken.

While the Market charges extra fees when too much is minted, the fees aren’t deposited as collateral in the Treasury, and this added complexity increases the likelihood of bugs. In pursuit of simplicity, we’d recommend disallowing minting excess beyond _maxBaseInBeforeSystemStabilityMode and only allowing minting up to the maximum amount. Currently, the minted amount is only bound if the fTokenMintPausedInStabilityMode is explicitly enabled but not by default. This has the added benefit of making validation and testing/verification easier to perform.

Rather than having the Treasury validate that it is solvent, the validation is performed in FxUSD (which requires calling the Market and Treasury). This makes the composability of the system fragile as modifying a contract locally can have far reaching consequences that are not apparent in the scope of the diff. Important validations related to core invariants should be clearly documented and be performed as close as possible to the component they are relevant to.

Rather than validating that the mint cap of fToken is reached in the FractionalToken implementation, it is done in FxUSD. This does not consider when FxUSD has not been added to the Market and the validation should be done in the token implementation instead. In general, performing validations against other contracts’ state variables instead of having the contract maintain the invariant is error prone.

**Recommendations**

Short term, validate that the collateral ratio has increased for minting xToken and redeeming fToken and that minting fToken and redeeming XToken do not put the collateral ratio below that stability ratio, allowing for immediate liquidations. While certain configurations of fees may be sufficient to prevent unexpected behavior, it is more robust to simply prohibit actions which have undesired effects.

Long term, perform validations specific to a component locally rather than sporadically performing validations in separate components. Simplify the architecture by refactoring disjointed components and make each component strictly validate itself. Create specifications for each component and their interactions, and perform invariant testing to ensure the specification matches the implementation.

## depositIntoPool function allows deposits during ongoing challenges

**Description**

The depositIntoPool function allows depositing tokens even when the assertion was already created or there are already more tokens than required to create the assertion.

In the best case scenario, where the assertion created wins and all the tokens are returned, no users would lose tokens. However, tokens deposited when the challenge is ongoing would allow users’ who previously deposited tokens that are now locked in the challenge to withdraw their tokens from the new depositor’s tokens.

**Exploit Scenario**

Alice deposits tokens into the pool, unaware that there is already an ongoing challenge. Eve, who deposited tokens that are now locked in the challenge, withdraws her tokens from Alice’s newly deposited tokens. The challenge ends up losing and Alice unknowingly loses her tokens, instead of Eve.

**Recommendations**

Short term, in the depositIntoPool function, modify the code to validate that the assertion is not already created and that the current contract’s token balance is less than what is required to create an assertion.

Long term, when designing a contract that holds users’ tokens, minimize the attack surface as muchaspossible by not accepting deposits after a target threshold has been reached.

## Lack of validation of mini stakes configuration

**Description**

The mini stake amounts set during initialization are not validated to decrease for each level (from block to single-step execution) despite that expectation being stated in the Arbitrum Improvement Proposal related to BOLD:

**Challenge-bonds, per level:** 3600/1000/100/10 ETH- required from validators to open challenges against an assertion observed on    Ethereum, for each level. Note that “level” corresponds to the level    of granularity at which the interactive dissection game gets played    over, starting at the block level, moving on to a range of WASM    execution steps, and then finally to the level of a single step of    execution.

The staking configuration is important to disincentivize fraud and make it feasible for an honest validator to meet the capital requirements for defending assertions.

**Recommendations**

Short term, modify the code to validate that the amount of stake required for each level decreases as the challenge progresses.

Long term, enforce expectations in code to prevent misconfigurations that have important ramifications on the protocol’s incentives.

## Potential token incompatibilities in staking pool

**Description**

The assertion staking pool is not currently reusable and transfers its entire allowance to the rollup for the requiredStake amount. In light of this, the use of the safeIncreaseAllowance function does not pose incompatibilities with tokens such as USDT, which require the current allowance to be zero when calling the approve function. However, it may be desirable to support reusable assertion staking pools in the future, and this subtlety should be considered in that event.

**Recommendations**

Short term, document the potential incompatibility and ensure that each transfer uses the entire allowance.

Long term, use the forceApprove function if the contract is revised to be reusable.

## Lack of zero-value checks in setter functions

**Description**

Certain functions fail to validate incoming arguments, so callers of these functions could mistakenly set important state variables to a zero value, misconfiguring the system.

For example, the Distribution_init function in the Distribution contract sets the depositToken , l1Sender , and splitter variables to the addresses passed as arguments without checking whether any of the values are the zero address. This may result in undefined behavior in the system.

In addition to the above, the following setter functions also lack zero-value checks:

  ● Distribution
  ○ editPool
  ● L1Sender
  ○ setRewardTokenConfig
  ● L2MessageReceiver
  ○ setParams
  ● RewardClaimer
  ○ RewardClaimer__init
  ○ setSplitter
  ○ setL1Sender

**Exploit Scenario**

Alice deploys a new version of the Distribution contract. When she invokes Distribution_init to set the contract’s parameters, she mistakenly enters a zero value, thereby misconfiguring the system.

**Recommendations**

Short term, add zero-value checks to all function arguments to ensure that callers cannot set incorrect values and misconfigure the system.

Long term, use the Slither static analyzer to catch common issues such as this one. Consider integrating a Slither scan into the project’s CI pipeline, pre-commit hooks, or build scripts.

## Risk of token loss due to lack of zero-value check of destination address

**Description**

We identified two functions that fail to verify whether the destination address is the zero address before sending tokens. This can lead to the unintentional burning of tokens.

First, as shown in figure 3.1, the claim function in the Distribution contract takes a receiver address, which is user-provided, and subsequently uses it to send native tokens via the sendMintMessage function in the L1Sender contract. Consequently, if the user mistakenly provides the zero address as the receiver argument, the tokens will inadvertently be burnt.

Second, the claim function in the RewardClaimer contract does not verify whether the receiver_ argument is the zero address; tokens will also be burned if a user supplies the zero address for this argument.

**Exploit Scenario**

Alice invokes the claim function in the Distribution contract to claim reward tokens. However, by mistake she supplies the zero address as the receiver address. Consequently, the claimed rewards are transmitted to the zero address, causing the tokens to be irreversibly lost.

**Recommendations**

Short term, add zero-address checks to the function arguments to ensure that callers cannot set incorrect values that result in the loss of tokens.

Long term, use the Slither static analyzer to catch common issues such as this one. Consider integrating a Slither scan into the project’s CI pipeline, pre-commit hooks, or build scripts.

## Missing zero-address checks in constructors

**Description**

None of the constructors in the various oracle contracts validate that their address arguments do not equal the zero address. As a result, important immutable state variables might be set to the zero address during deployment, effectively making the given contract unusable and requiring a redeployment.

**Recommendations**

Short term, add a check to each constructor to ensure that each address argument does not equal the zero address.

Long term, use the Slither static analyzer to catch common issues such as this one. Consider integrating a Slither scan into the project’s CI pipeline, pre-commit hooks, or build scripts.

## Lack of validation of Chainlink price feed answers

**Description**

The validation of the price returned by Chainlink is incomplete, which means that incorrect prices could be used in the protocol. This could lead to loss of funds or otherwise cause internal accounting errors that might break the correct functioning of the protocol.

Because the Chainlink-returned price is of type int256 , the following two scenarios could happen:

 - ● The price feed answer could be a negative integer. First off, this
   is highly unlikely for the particular price feeds used by f(x).
   However, if a negative integer is returned, it will be unsafely cast
   to an unsigned integer ( uint256 ) on line 57 of
   _readSpotPriceByChainlink . This will likely lead to a revert because the unsigned value of a cast signed negative integer will likely be
   very high, but it might also lead to the use of an incorrect price. 
   ●A Chainlink price feed can also return zero as the answer. In this
   case, the isValid Boolean will be set to false , which will ensure
   the incorrect price is not actually used, as shown in figure 3.2.

**Exploit Scenario**

The Chainlink price feed returns a negative price, which when cast to an unsigned integer is considered valid. As a result, an incorrect price is used.

**Recommendations**

Short term, add a check inside the _readSpotPriceByChainlink function that ensures answer is greater than 0.

Long term, add validation of returned results from all external sources.

## Lack of validation of updates to system configuration parameters

**Description**

Several configuration functions (figures 4.1–4.3) do not validate that updates to configuration parameters actually result in a change in value. Although setting a parameter to its current value is benign, it may obscure a logical error in a peripheral program that would be readily identifiable if the update were to revert and raise an alarm.

**Recommendations**

Short term, add validation to these functions to require that the new value is not equal to the previous value.

Long term, add validation to all configuration functions to ensure they either perform a configuration state update or cause a revert.

## Denial-of-service conditions caused by the use of more than 256 slices

**Description**

The owner of a Proteus-based automated market maker (AMM) can update the system parameters to cause a denial of service (DoS) upon the execution of swaps, withdrawals, and deposits.

The Proteus AMM engine design supports the creation of an arbitrary number of slices. Slices are used to segment an underlying bonding curve and provide variable liquidity across that curve. The owner of a Proteus contract can update the number of slices by calling the _updateSlices function at any point.

When a user requests a swap, deposit, or withdrawal operation, the Proteus contract first calls the _findSlice function (figure 1.1) to identify the slice in which it should perform the operation. The function iterates across the slices array and returns the index, i, of the slice that has the current ratio of token balances, m.

However, the index, i, is defined as a uint8. If the owner sets the number of slices to at least 257 (by calling _updateSlices) and the current m is in the 257th slice, i will silently overflow, and the while loop will continue until an out-of-gas (OOG) exception occurs. If a deposit, withdrawal, or swap requires the 257th slice to be accessed, the operation will fail because the _findSlice function will be unable to reach that slice.

**Exploit Scenario**

Eve creates a seemingly correct Proteus-based primitive (one with only two slices near the asymptotes of the bonding curve). Alice deposits assets worth USD 100,000 into a pool. Eve then makes a deposit of X and Y tokens that results in a token balance ratio, m, of 1. Immediately thereafter, Eve calls _updateSlices and sets the number of slices to 257, causing the 256th slice to have an m of 1.01. Because the current m resides in the 257th slice, the _findSlice function will be unable to find that slice in any subsequent swap, deposit, or withdrawal operation. The system will enter a DoS condition in which all future transactions will fail.

If Eve identifies an arbitrage opportunity on another exchange, Eve will be able to call _updateSlices again, use the unlocked curve to buy the token of interest, and sell that token on the other exchange for a pure profit. Effectively, Eve will be able to steal user funds.

**Recommendations**

Short term, change the index, i, from the uint8 type to uint256; alternatively, create an upper limit for the number of slices that can be created and ensure that i will not overflow when the _findSlice function searches through the slices array.

Long term, consider adding a delay between a call to _updateSlices and the time at which the call takes effect on the bonding curve. This will allow users to withdraw from the system if they are unhappy with the new parameters. Additionally, consider making slices immutable after their construction; this will significantly reduce the risk of undefined behavior.

## Ocean may accept unexpected airdrops

**Description**

Unexpected transfers of tokens to theOceancontractmay break its internal accounting, essentially leading to the loss of the transferred asset. To mitigate this risk,Oceanattempts to reject airdrops.

Per the ERC721 and ERC1155 standards, contracts must implement specific methods to accept or deny token transfers. To do this, theOceancontract uses the onERC721ReceivedandonERC1155Receivedcallbacks and _ERC1155InteractionStatusand_ERC721InteractionStatusstorage flags. These storage flags are enabled in ERC721 and ERC1155 wrapping operations to facilitate successful standard-compliant transfers.

However, the_erc721Unwrapand_erc1155Unwrapfunctionsalso enable the _ERC721InteractionStatusand_ERC1155InteractionStatusflags, respectively. Enabling these flags allows for airdrops, since theOceancontract, not the user, is the recipient of the tokens in unwrapping operations.

**Exploit Scenario**

Alice calls the _erc721Unwrap function. When the onERC721Received callback function in Alice’s contract is called, Alice mistakenly sends the ERC721 tokens back to the Ocean contract. As a result, her ERC721 is permanently locked in the contract and effectively burned.

**Recommendations**

Short term, disallow airdrops of standard-compliant tokens during unwrapping interactions and document the edge cases in which the Ocean contract will be unable to stop token airdrops.

Long term, when the Ocean contract is expecting a specific airdrop, consider storing the originating address of the transfer and the token type alongside the relevant interaction f lag.


