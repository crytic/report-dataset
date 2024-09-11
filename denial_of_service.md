## Attackers can prevent the pool manager from finishing liquidation

**Description**

The finishCollateralLiquidation function requires that a liquidation is no longer active. However, an attacker can prevent the liquidation from finishing by sending a minimal amount of collateral token to the liquidator address.

The finishCollateralLiquidation function uses the isLiquidationActive function to verify if the liquidation process is finished by checking the collateral asset balance of the liquidator address. Because anyone can send tokens to that address, it is possible to make isLiquidationActive always return false.

**Exploit Scenario**

Alice's loan is being liquidated. Bob, the pool manager, tries to call finishCollateralLiquidation to get back the recovered funds. Eve front-runs Bob’s call by sending 1 token of the collateral asset to the liquidator address. As a consequence, Bob cannot recover the funds.

**Recommendations**

Short term, use a storage variable to track the remaining collateral in the Liquidator contract. As a result, the collateral balance cannot be manipulated through the transfer of tokens and can be safely checked in isLiquidationActive.

Long term, avoid using exact comparisons for ether and token balances, as users can increase those balances by executing transfers, making the comparisons evaluate to false.

## Unbounded loop can cause denial of service

**Description**

Under certain conditions, the withdrawal code will loop, permanently blocking users from getting their funds.

The beforeWithdraw function runs before any withdrawal to ensure that the vault has sufficient assets. If the vault reserves are insufficient to cover the withdrawal, it loops over each strategy, incrementing the _strategyId pointer value with each iteration, and withdrawing assets to cover the withdrawal amount.

However, during an iteration, if the vault raises enough assets that the amount needed by the vault becomes zero or that the current strategy no longer has assets, the loop would keep using the same strategyId until the transaction runs out of gas and fails, blocking the withdrawal.

**Exploit Scenario**

Alice tries to withdraw funds from the protocol. The contract may be in a state that sets the conditions for the internal loop to run indefinitely, resulting in the waste of all sent gas, the failure of the transaction, and blocking all withdrawal requests.

**Recommendations**

Short term, add logic to increment the _strategyId variable to point to the next strategy in the StrategyQueue before the continue statement. Long term, use unit tests and fuzzing tools like Echidna to test that the protocol works as expected, even for edge cases.

## Harvest operation could be blocked if eligibility check on a strategy reverts

**Description**

During harvest, if any of the strategies in the queue were to revert, it would prevent the loop from reaching the end of the queue and also block the entire harvest operation.

When the harvest function is executed, a loop iterates through each of the strategies in the strategies queue, and the canHarvest() check runs on each strategy to determine if it is eligible for harvesting; if it is, the harvest logic is executed on that strategy.

However, if the canHarvest() check on a particular strategy within the loop reverts, external calls from the canHarvest() function to check the status of rewards could also revert. Since the call to canHarvest() is not inside of a try block, this would prevent the loop from proceeding to the next strategy in the queue (if there is one) and would block the entire harvest operation.

Additionally, within the harvest function, the runHarvest function is called twice on a strategy on each iteration of the loop. This could lead to unnecessary waste of gas and possibly undefined behavior.

**Recommendations**

Short term, wrap external calls within the loop in try and catch blocks, so that reverts can be handled gracefully without blocking the entire operation. Additionally, ensure that the canHarvest function of a strategy can never revert.

Long term, carefully audit operations that consume a large amount of gas, especially those in loops. Additionally, when designing logic loops that make external calls, be mindful as to whether the calls can revert, and wrap them in try and catch blocks when necessary.

## Risk of DoS attacks due to rate limits

**Description**

Due to the rate limits imposed on MONO deposit and withdrawal operations, malicious users could launch denial-of-service (DoS) attacks.

The PSM contract extends the TimeBasedRateLimiter contract, which defines the maximum number of MONO tokens that can be minted and redeemed within a preset duration window. This allows the protocol to manage the inflow and outflow of assets (figures 1.1 and 1.2).

However, a dedicated adversary could deposit enough collateral assets to reach the minting limit and immediately withdraw enough MONO to reach the redeeming limit in the same transaction; this would exhaust the limits of a given duration window, preventing all subsequent users from entering and exiting the system. Additionally, if the duration is long (e.g., a few hours or a day), such attacks would cause the PSM contract to become unusable for extended periods of time. It is important to note that adversaries conducting such attacks would incur the fee imposed on redemption operations, making the attack less appealing.

**Exploit Scenario**

Alice, the guardian, sets resetMintDuration and resetRedeemDuration to one day. As soon as the minting and redeeming limits reset, Eve deposits a large amount of USDC for MONO and immediately burns the MONO for USDC. This transaction maxes out the minting and redeeming limits, preventing users from performing deposits and withdrawals for an entire day.

**Recommendations**

Short term, set the resetMintDuration and resetRedeemDuration variables to reasonably small values to mitigate the impact of DoS attacks. Note that setting them to zero would defeat the purpose of rate limiting because the limits will be reset on every minting or redeeming operation.

Long term, implement off-chain monitoring to detect unusual user interactions, and develop an incident response plan to ensure that any issues that arise can be addressed promptly and without confusion.

## Incorrect calculation in getPositionRepartition can lock a user’s position

**Description**

A lender can lock their position if they deposit using (or update their position rate to equal) a rate that was borrowed in the previous lending cycle, but ends up not being borrowed during the next cycle.

The protocol uses interest rates and epochs to determine the order in which assets will be borrowed from during a lending cycle. The deposit function in RCLLenders is designed to account for this by updating different state variables depending on the rate and the epoch in which the assets were deposited.

This function internally uses the deposit function from TickLogic to perform the accounting

However, if the rate was borrowed during the last lending cycle but does not end up being borrowed during the next lending cycle, the protocol will incorrectly consider the position as part of the new epoch based on the position.epochId.

Since the protocol will include the value in the base epoch, the epoch.deposited value will not be updated. This will cause the getPositionRepartition function to incorrectly calculate the position’s unborrowed and borrowed amount, causing a division-by-zero error because epoch.deposited is equal to 0

This same issue can occur if a user updates the rate of their position to a previously borrowed rate that ends up not being borrowed in the next lending cycle. This is due to the updateRate function updating the position.epochId.

In both cases, the user will be prevented from withdrawing their position, essentially locking up their assets until the borrower borrows from their position or repays the current loan.

**Exploit Scenario**

A Revolving Credit Line is created for Alice with a maximum borrowable amount of 10 ether.

Bob deposits 5 ether into the pool at a 10% rate, and Charlie deposits 10 ether into the pool at a 15% rate. Alice borrows 10 ether and, after some time has passed, she repays the loan.

Bob updates the rate of his position to 5% but later changes his mind and updates it back to 10%. Charlie sees this and updates his rate to 9% so his position ends up being fully borrowed.

Alice borrows 10 ether for the next loan cycle. Bob’s position is left unborrowed and he attempts to withdraw; however, this reverts with a division-by-zero error.

Since Bob’s position was accounted for in the base epoch, but the position.epochId was updated as if it was in the new epoch due to updateRate, the system accounting breaks for his position. Bob is prevented from performing any action on his position until his position is borrowed or the loan is repaid.

**Recommendations**

Short term, consider updating the getPositionRepartition function to correctly account for positions that are deposited into, or whose rate was updated to, a previously borrowed rate. This could be done by adding a case where position.epochId == tick.currentEpochId. Ensure that the code is properly tested after this change.

Long term, consider simplifying the way epochs are accounted for in the system. Improve unit test coverage and create a list of system properties that can be tested with smart contract fuzzing.

## Detached positions cannot be exited during subsequent loans

**Description**

If a position has been detached during a previous loan cycle and is borrowed in a subsequent loan cycle, that position cannot be exited.

The protocol allows positions that can be fully matched to exit the loan, withdrawing the full amount of their deposit and the accumulated accruals. However, a position that has been detached during a previous lending cycle will not be able to exit; instead, the function will revert with an arithmetic underflow:

Due to this error, the lender will be prevented from exiting their position and will have to wait for the loan to be repaid to withdraw their assets.

**Exploit Scenario**

A Revolving Credit Line is created for Alice with a maximum borrowable amount of 100 ether and a duration of 10 weeks.

1. Bob deposits 20 ether into the pool at a 10% rate, and Alice borrows 10 ether.
2. Bob assumes Alice will not borrow any more assets and detaches his position so he can utilize the assets elsewhere. He receives 10 ether from the position.
3. Alice repays the loan and again borrows 10 ether.
4. Charlie deposits 10 ether into the pool at a 10% rate.
5. Bob decides to exit his position. However, when he executes the exit function, it reverts with an arithmetic underflow

**Recommendations**

Short term, consider revising how detached positions are considered in the system during multiple loan cycles.

Long term, improve unit test coverage and create a list of system properties that can be tested with smart contract fuzzing. For example, this issue could have been discovered by implementing a property test that checks that calling exit does not revert if all the preconditions are met

## Interest rates can become extreme, allowing denial-of-service attacks

**Description**

The Ajna protocol calculates interest rates based on the deviation of the meaningful actual utilization (MAU) from the target utilization (TU). If the MAU is sufficiently higher than the TU, then the interest rate will increase, as there is more demand for lending; interest will decrease if the TU is less than the MAU. The TU is calculated as the exponential moving average (EMA) of the collateralization ratio per pool debt, while the MAU is calculated as the average price of a loan weighted by the debt.

As a result, when a pool contains much more collateral than deposited assets, the pool's interest rate will rise, even if debt is very low. Without sufficient debt, new lenders are not incentivized to deposit, even as interest rates grow to be very high. Borrowers are incentivized to repay debt as rates increase, but they are not directly incentivized to withdraw their collateral. Lender rates will continue to rise while the pool is in this state, eventually denying service to the pool.

There are many ways for rates to reach extreme highs or lows. Generally, market effects will keep rates in check, but if they do not, the rates could reach extreme values. Once loans are repaid and there is no more activity, there is no mechanism to cause the rates to return to normal.

The protocol could try to affect the rates by adding deposits and loans directly. But if the pool were under attack, the mitigation efforts could be countered by an attacker adding deposits, loans, or collateral themselves.

**Exploit Scenario**

Eve wants to prevent a shorting market from developing for her $EVE token. She takes the following actions:

 - ● Creates a new EVE/USDC pool where EVE is the quote token
  ● Deposits
   20 EVE into the bucket of price 1
   ● Provides 1,000 USDC as collateral
  ● Borrows 10 EVE
  ● Triggers interest rate increases every 12 hours
  ● Begins marketing her EVE token two months later when the interest
   rate is over 2,000%

As a result, would-be short sellers are deterred by high-interest rates, and the token is unable to be economically shorted, effectively disabling the pool.

**Recommendations**

Short term, consider using a nonlinear change in interest rates such that, as rates get higher (or lower) compared to the initial rate, a smaller increment can be used. Another way to mitigate this issue would be to implement some sort of interest rate “reset” that can be triggered under certain conditions or by a permissioned function.

Long term, improve the protocol’s unit test coverage to handle edge cases and ensure that the protocol behaves as intended. In addition, come up with a variety of edge cases and unlikely scenarios to adequately stress test the system.

## Tokens may be trapped in an invalid position

**Description**
The Raft finance contracts allow users to take out positions by depositing collateral and minting a corresponding amount of R tokens as debt. In order to exit a position, a user must pay back their debt, which allows them to receive their collateral back. To check that a position is closed, the _managePosition function contains a branch that validates that the position's debt is zero after adjustment. However, if the position's debt is zero but there is still some collateral present even after adjustment, then the position is considered invalid and cannot be closed. This could be problematic, especially if some dust is present in the position after the collateral is withdrawn.

**Exploit Scenario**

Alice, a borrower, wants to pay back her debt and receive her collateral in exchange. However, she accidentally leaves some collateral in her position despite paying back all her debt. As a result, her position cannot be closed.

**Recommendations**

Short term, if a position's debt is zero, have the _managePosition function refund any excess collateral and close the position.

Long term, carefully investigate potential edge cases in the system and use smart contract fuzzing to determine if those edge cases can be realistically reached.

## Withdrawal queue can be forcibly activated to hinder bridge operation

**Description**

The withdrawal queue can be forcibly activated to impede the proper operation of the bridge.

The RootERC20PredicateFlowRate contract implements a withdrawal queue to more easily detect and stop large withdrawals from passing through the bridge (e.g., bridging illegitimate funds from an exploit). A transaction can enter the withdrawal queue in four ways:
1. If a token’s flow rate has not been configured by the rate control admin
2. If the withdrawal amount is larger than or equal to the large transfer threshold for that token
3. If, during a predefined period, the total withdrawals of that token are larger than the defined token capacity
4. If the rate controller manually activates the withdrawal queue by using the activateWithdrawalQueue function

In cases 3 and 4 above, the withdrawal queue becomes active for all tokens, not just the individual transfers. Once the withdrawal queue is active, all withdrawals from the bridge must wait a specified time before the withdrawal can be finalized. As a result, a malicious actor could withdraw a large amount of tokens to forcibly activate the withdrawal queue and hinder the expected operation of the bridge.

**Exploit Scenario 1**

Eve observes Alice initiating a transfer to bridge her tokens back to the mainnet. Eve also initiates a transfer, or a series of transfers to avoid exceeding the per-transaction limit, of sufficient tokens to exceed the expected flow rate. With Alice unaware she is being targeted for griefing, Eve can execute her withdrawal on the root chain first, cause Alice’s withdrawal to be pushed into the withdrawal queue, and activate the queue for every other bridge user.

**Exploit Scenario 2**

Mallory has identified an exploit on the child chain or in the bridge itself, but because of the withdrawal queue, it is not feasible to exfiltrate the funds quickly enough without risking getting caught. Mallory identifies tokens with small flow rate limits relative to their price and repeatedly triggers the withdrawal queue for the bridge, degrading the user experience until Immutable disables the withdrawal queue. Mallory takes advantage of this window of time to carry out her exploit, bridge the funds, and move them into a mixer.

**Recommendations**

Short term, explore the feasibility of withdrawal queues on a per-token basis instead of having only a global queue. Be aware that if the flow rates are set low enough, an attacker could feasibly use them to grief all bridge users.

Long term, develop processes for regularly reviewing the configuration of the various token buckets. Fluctuating token values may unexpectedly make this type of griefing more feasible.

## Panicking checked arithmetic may prevent deposits and withdraws to rebalance pool

**Description**

The rebalance pool’s deposit and withdraw functionality computes the latest balance of a user’s account at each interaction. This calculation strictly assumes that the _newBalance amount is less than the old, but does not validate that this is the case. If this is ever not the case, the pool will revert due to Solidity’s built-in checked arithmetic, and the user will not be able to deposit or withdraw.

The following check should resemble _balance.amount > _newBalance in order to avoid reverting on the overflow case and emit an event only when a real loss occurs (i.e., the user’s balance has decreased as indicated by the UserDepositChange event’s loss parameter).

**Exploit Scenario**

A user deposits to the rebalance pool and is unable to withdraw because an overflow causes the transaction to revert.

**Recommendations**

Short term, validate that _balance.amount > _newBalance is true before unconditionally performing checked arithmetic during interactions that users must be able to successfully complete.

Long term, review the logical and arithmetic operations for incorrect assumptions and perform fuzz testing to identify corner cases like these.

## Reverting when minting xToken can prevent re-collateralization

**Description**

When the treasury processes the mintXToken action, it validates that the oracle price is valid, and, if invalid, it reverts. During stability mode (collateral ratio between 100% and 130%), the protocol is designed to recover by incentivizing the minting of xToken in order to re-collateralize above 130%. Minting xToken may fail, even though it is necessary to recover from stability mode, and cause the protocol to collapse. Instead, the protocol could always permit the minting of xToken during stability mode and handle invalid prices by conservatively pricing the collateral and favoring the protocol’s credit risk health at the expense of the user minting.

**Recommendations**

Short term, consider allowing minting of xToken at a conservative price when the protocol is in stability mode.

Long term, identify actions that should never fail to succeed, such as those relevant to re-collateralizing the protocol, and implement invariant testing that ensures that they always succeed.


