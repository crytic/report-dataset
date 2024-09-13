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
