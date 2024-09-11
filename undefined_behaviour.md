## Solidity compiler optimizations can be problematic



  

**Description**

The Increment Protocol contracts have enabled optional compiler optimizations in Solidity.

  

There have been several optimization bugs with security implications. Moreover, optimizations are actively being developed. Solidity compiler optimizations are disabled by default, and it is unclear how many contracts in the wild actually use them. Therefore, it is unclear how well they are being tested and exercised.

  

Security issues due to optimization bugs have occurred in the past. A medium- to high-severity bug in the Yul optimizer was introduced in Solidity version 0.8.13 and was fixed only recently, in Solidity version 0.8.17. Another medium-severity optimization bug—one that caused memory writes in inline assembly blocks to be removed under certain conditions—was patched in Solidity 0.8.15.

  

A compiler audit of Solidity from November 2018 concluded that the optional optimizations may not be safe.

  

It is likely that there are latent bugs related to optimization and that new bugs will be introduced due to future optimizations.

  

**Exploit Scenario**

A latent or future bug in Solidity compiler optimizations causes a security vulnerability in the Increment Protocol contracts.

  

**Recommendations**

Short term, measure the gas savings from optimizations and carefully weigh them against the possibility of an optimization-related bug.

  

Long term, monitor the development and adoption of Solidity compiler optimizations to assess their maturity.

  
  

## Support for multiple reserve tokens allows for arbitrage

  

**Description**

Because the UA token contract supports multiple reserve tokens, it can be used to swap one reserve token for another at a ratio of 1:1. This creates an arbitrage opportunity, as it enables users to swap reserve tokens with different prices.

  

Users can deposit supported reserve tokens in the UA contract in exchange for UA tokens at a 1:1 ratio (figure 4.1)

  

Similarly, users can withdraw the amount of a deposit by returning their UA in exchange for any supported reserve token, also at a 1:1 ratio (figure 4.2).

  

Thus, a user could mint UA by depositing a less valuable reserve token and then withdraw the same amount of a more valuable token in one transaction, engaging in arbitrage.

  

**Exploit Scenario**

Alice, who holds the governance role, adds USDC and DAI as reserve tokens. Eve notices that DAI is trading at USD 0.99, while USDC is trading at USD 1.00. Thus, she decides to mint a large amount of UA by depositing DAI and to subsequently return the DAI and withdraw USDC, allowing her to make a risk-free profit.

  

**Recommendations**

Short term, document all front-running and arbitrage opportunities in the protocol to ensure that users are aware of them. As development continues, reassess the risks associated with those opportunities and evaluate whether they could adversely affect the protocol.

  

Long term, implement an off-chain monitoring solution (like that detailed in TOB-INC-13) to detect any anomalous fluctuations in the prices of supported reserve tokens. Additionally, develop an incident response plan to ensure that any issues that arise can be addressed promptly and without confusion. (See appendix D for additional details on creating an incident response plan.)

  
  
  

## Problematic use of primitive operations on fixed-point integers

  

**Description**

The protocol’s use of primitive operations over fixed-point signed and unsigned integers increases the risk of overflows and undefined behavior.

  

The Increment Protocol uses the PRBMathSD59x18 and PRBMathUD60x18 math libraries to perform operations over 59x18 signed integers and 60x18 unsigned integers, respectively (specifically to perform multiplication and division and to find their absolute values). These libraries aid in calculations that involve percentages or ratios or require decimal precision.

  

When a smart contract system relies on primitive integers and fixed-point ones, it should avoid arithmetic operations that involve the use of both types. For example, using x.wadMul(y) to multiply two fixed-point integers will provide a different result than using x * y. For that reason, great care must be taken to differentiate between variables that are fixed-point and those that are not. Calculations involving fixed-point values should use the provided library operations; calculations involving both fixed-point and primitive integers should be avoided unless one type is converted to the other.

  

However, a number of multiplication and division operations in the codebase use both primitive and fixed-point integers. These include those used to calculate the new time-weighted average prices (TWAPs) of index and market prices (figure 8.1).

  

Similarly, the _getUnrealizedPnL function in the Perpetual contract calculates the tradingFees value by multiplying a primitive and a fixed-point integer (figure 8.2).

  

These calculations can lead to unexpected overflows or cause the system to enter an undefined state. Note that there are other such calculations in the codebase that are not documented in this finding.

  

**Recommendations**

Short term, identify all state variables that are fixed-point signed or unsigned integers. Additionally, ensure that all multiplication and division operations involving those state variables use the wadMul and wadDiv functions, respectively. If the Increment Finance team decides against using wadMul or wadDiv in any of those operations (whether to optimize gas or for another reason), it should provide inline documentation explaining that decision.

## Incorrect argument passed to _getPlatformOriginationFee

**Description**

The getOriginationFees function incorrectly uses msg.sender instead of the loan_ parameter. As a result, it returns an incorrect result to users who want to know how much a loan is paying in origination fees.

**Exploit Scenario**

Bob, a borrower, wants to see how much his loan is paying in origination fees. He calls getOriginationFees but receives an incorrect result that does not correspond to what the loan actually pays.

**Recommendations**

Short term, correct the getOriginationFees function to use loan_ instead of msg.sender.

Long term, add tests for view functions that are not used inside the protocol but are intended for the end-users.

## Incorrect implementation of EIP-4626

**Description**

The Pool implementation of EIP-4626 is incorrect for maxDeposit and maxMint because these functions do not consider all possible cases in which deposit or mint are disabled.

EIP-4626 is a standard for implementing tokenized vaults. In particular, it specifies the following:
● maxDeposit: MUST factor in both global and user-specific limits. For example, if deposits are entirely disabled (even temporarily), it MUST return 0.
● maxMint: MUST factor in both global and user-specific limits. For example, if mints are entirely disabled (even temporarily), it MUST return 0.

The current implementation of maxDeposit and maxMint in the Pool contract directly call and return the result of the same functions in PoolManager (figure 6.1). As shown in figure 6.1, both functions rely on _getMaxAssets, which correctly checks that the liquidity cap has not been reached and that deposits are allowed and otherwise returns 0. However, these checks are insufficient.

The deposit and mint functions have a checkCall modifier that will call the canCall function in the PoolManager to allow or disallow the action. This modifier first checks if the global protocol pause is active; if it is not, it will perform additional checks in _canDeposit. For this issue, it will be impossible to deposit or mint if the Pool is not active.

The maxDeposit and maxMint functions should return 0 if the global protocol pause is active or if the Pool is not active; however, these cases are not considered.

**Exploit Scenario**

A third-party protocol wants to deposit into Maple’s pool. It first calls maxDeposit to obtain the maximum amount of asserts it can deposit and then calls deposit. However, the latter function call will revert because the protocol is paused.

**Recommendations**

Short term, return 0 in maxDeposit and maxMint if the protocol is paused or if the pool is not active.

Long term, maintain compliance with the EIP specification being implemented (in this case, EIP-4626).

## setAllowedSlippage and setMinRatio functions are unreachable

**Description**

The administrative functions setAllowedSlippage and setMinRatio have a requirement that they can be called only by the poolManager. However, they are not called by any reachable function in the PoolManager contract.

**Exploit Scenario**

Alice, a pool administrator, needs to adjust the slippage parameter of a particular collateral token. Alice’s transaction reverts since she is not the poolManager contract address. Alice checks the PoolManager contract for a method through which she can set the slippage parameter, but none exists.

**Recommendations**

Short term, add functions in the PoolManager contract that can reach setAllowedSlippage and setMinRatio on the LoanManager contract.

Long term, add unit tests that validate all system parameters can be updated successfully.

## Pool owners may not receive BPT payouts

**Description**

The _collectAumManagementFees function collects assets under management (AUM) fees and divides them between the protocol and the pool owner. These fees are paid first to the protocol and then to the pool owner; however, after the protocol is paid, there may not be any fees left for the pool owner.

As shown in figure 1.1, the _collectAumManagementFees function will immediately return 0 if the BPT amount is too low. If the calculated protocol fee is exactly equivalent to the BPT amount, the managerBPTAmount will be 0.

**Exploit Scenario**

The Balancer system is initialized with a cached protocol fee percentage of 10. Alice, a privileged user, adds a new token with an initial total supply of 1 to the pool. The management fee calculation determines that the Balancer protocol will receive 10 BPT tokens and that the pool owner will receive 0.

**Recommendations**

Short term, if the _collectAumManagementFees function is behaving as expected, clearly document the expectations surrounding the function; if it is not, bind the minimum BPT amount to at least the cached protocol fee percentage.

Long term, analyze all incoming arguments and evaluate whether their bounds are safe.

## Incorrect application of penalty fee rate

**Description**

A Fraxlend pair can have a maturity date, after which a penalty rate is applied to the interest to be paid by the borrowers. However, the penalty rate is also applied to the amount of time immediately before the maturity date.

As shown in figure 3.1, the _addInterest function checks whether a pair is past maturity. If it is, the function sets the new rate (the _newRate parameter) to the penalty rate (the penaltyRate parameter) and then uses it to calculate the matured interest. The function should apply the penalty rate only to the time between the maturity date and the current time; however, it also applies the penalty rate to the time between the last interest accrual (_deltaTime) and the maturity date, which should be subject only to the normal interest rate.

**Exploit Scenario**

A Fraxlend pair’s maturity date is 100, the delta time (the last time interest accrued) is 90, and the current time is 105. Alice decides to repay her debt. The _addInterest function is executed, and the penalty rate is also applied to the 10 units of time between the last interest accrual and the maturity date. As a result, Alice owes more in interest than she should.

**Recommendations**

Short term, modify the associated code so that if the _isPastMaturity branch is taken and the _currentRateInfo.lastTimestamp value is less than maturityDate value, the penalty interest rate is applied only for the amount of time after the maturity date.

Long term, identify edge cases that could occur in the interest accrual process and implement unit tests and fuzz tests to validate them.

## FraxlendPairDeployer cannot deploy contracts of fewer than 13,000 bytes

**Description**

The FraxlendPairDeployer contract, which is used to deploy new pairs, does not allow contracts that contain less than 13,000 bytes of code to be deployed.

To deploy new pairs, users call the deploy or deployCustom function, which then internally calls _deployFirst. This function uses the create2 opcode to create a contract for the pair by concatenating the bytecode stored in contractAddress1 and contractAddress2.

The setCreationCode function, which uses solmate’s SSTORE2 library to store the bytecode for use by create2, splits the bytecode into two separate contracts (contractAddress1 and contractAddress2) if the _creationCode size is greater than 13,000.

The first problem is that if the _creationCode size is less than 13,000, BytesLib.slice will revert with the slice_outOfBounds error, as shown in figure 7.2.

Assuming that the first problem does not exist, another problem arises from the use of SSTORE2.read in the _deployFirst function (figure 7.3). If the creation code was less than 13,000 bytes, contractAddress2 would be set to address(0). This would cause the SSTORE2.read function’s pointer.code.length - DATA_OFFSET computation, shown in figure 7.4, to underflow, causing the SSTORE2.read operation to panic.

**Exploit Scenario**

Bob, the FraxlendPairDeployer contract’s owner, wants to set the creation code to be a contract with fewer than 13,000 bytes. When he calls setCreationCode, it reverts.

**Recommendations**

Short term, make the following changes:
● In setCreationCode, in the line that sets the _firstHalf variable, replace 13000 in the third argument of BytesLib.slice with min(13000, _creationCode.length).
● In _deployFirst, add a check to ensure that the SSTORE2.read(contractAddress2) operation executes only if contractAddress2 is not address(0).

Alternatively, document the fact that it is not possible to deploy contracts with fewer than 13,000 bytes.

Long term, improve the project’s unit tests and fuzz tests to check that the functions behave as expected and cannot unexpectedly revert.

## setCreationCode fails to overwrite _secondHalf slice if updated code size is less than 13,000 bytes

**Description**

The setCreationCode function permits the owner of FraxlendPairDeployer to set the bytecode that will be used to create contracts for newly deployed pairs. If the _creationCode size is greater than 13,000 bytes, it will be split into two separate contracts (contractAddress1 and contractAddress2). However (assuming that TOB-FXLEND-7 were fixed), if a FraxlendPairDeployer owner were to change the creation code from one of greater than 13,000 bytes to one of fewer than 13,000 bytes, contractAddress2 would not be reset to address(0); therefore, contractAddress2 would still contain the second half of the previous creation code.

**Exploit Scenario**

Bob, FraxlendPairDeployer’s owner, changes the creation code from one of more than 13,000 bytes to one of less than 13,000 bytes. As a result, deploy and deployCustom deploy contracts with unexpected bytecode.

**Recommendations**

Short term, modify the setCreationCode function so that it sets contractAddress2 to address(0) at the beginning of the function.

Long term, improve the project’s unit tests and fuzz tests to check that the functions behave as expected and cannot unexpectedly revert.

## Uninformative implementation of maxDeposit and maxMint from EIP-4626

**Description**

The GVault implementation of EIP-4626 is uninformative for maxDeposit and maxMint, as they return only fixed, extreme values.

EIP-4626 is a standard to implement tokenized vaults. In particular, the following is specified:
● maxDeposit: MUST factor in both global and user-specific limits, like if deposits are entirely disabled (even temporarily) it MUST return 0. MUST return 2 256 - 1 if there is no limit on the maximum amount of assets that may be deposited.
● maxMint: MUST factor in both global and user-specific limits, like if mints are entirely disabled (even temporarily) it MUST return 0. MUST return 2  256 - 1 if there is no limit on the maximum amount of assets that may be deposited.

The current implementation of maxDeposit and maxMint in the GVault contract directly return the maximum value of the uint256 type
This implementation, however, does not provide any valuable information to the user and may lead to faulty integrations with third-party systems.

**Exploit Scenario**

A third-party protocol wants to deposit into a GVault. It first calls maxDeposit to know the maximum amount of asserts it can deposit and then calls deposit. However, the latter function call will revert because the value is too large.

**Recommendations**

Short term, return suitable values in maxDeposit and maxMint by considering the amount of assets owned by the caller as well any other global condition (e.g., a contract is paused).

Long term, ensure compliance with the EIP specification that is being implemented (in this case, EIP-4626).

## moveStrategy runs of out gas for large inputs

**Description**

Reordering strategies can trigger operations that will run out-of-gas before completion.

A GVault contract allows different strategies to be added into a queue. Since the order of them is important, the contract provides moveStrategy, a function to let the owner to move a strategy to a certain position of the queue.

The documentation states that if the position to move a certain strategy is larger than the number of strategies in the queue, then it will be moved to the tail of the queue. This implemented using the move function

However, if a large number of steps is used, the loop will never finish without running out of gas. A similar issue affects StrategyQueue.withdrawalQueue, if called directly.

**Exploit Scenario**

Alice creates a smart contract that acts as the owner of a GVault. She includes code to reorder strategies using a call to moveStrategy. Since she wants to ensure that a certain strategy is always moved to the end of the queue, she uses a very large value as the position. When the code runs, it will always run out of gas, blocking the operation.

**Recommendations**

Short term, ensure the execution of the move ends in a number of steps that is bounded by the number of strategies in the queue.

Long term, use unit tests and fuzzing tools like Echidna to test that the protocol works as expected, even for edge cases.

## Solidity compiler optimizations can be problematic

**Description**

The GSquared Protocol contracts have enabled optional compiler optimizations in Solidity.

There have been several optimization bugs with security implications. Moreover, optimizations are actively being developed. Solidity compiler optimizations are disabled by default, and it is unclear how many contracts in the wild actually use them. Therefore, it is unclear how well they are being tested and exercised.

Security issues due to optimization bugs have occurred in the past. A medium- to high-severity bug in the Yul optimizer was introduced in Solidity version 0.8.13 and was fixed only recently, in Solidity version 0.8.17. Another medium-severity optimization bug—one that caused memory writes in inline assembly blocks to be removed under certain conditions— was patched in Solidity 0.8.15.

A compiler audit of Solidity from November 2018 concluded that the optional optimizations may not be safe.

It is likely that there are latent bugs related to optimization and that new bugs will be introduced due to future optimizations.

**Exploit Scenario**

A latent or future bug in Solidity compiler optimizations causes a security vulnerability in the GSquared Protocol contracts.

**Recommendations**

Short term, measure the gas savings from optimizations and carefully weigh them against the possibility of an optimization-related bug.

Long term, monitor the development and adoption of Solidity compiler optimizations to assess their maturity.

## Solidity compiler optimizations can be problematic

**Description**

The Ondo protocol contracts have enabled optional compiler optimizations in Solidity.

There have been several optimization bugs with security implications. Moreover, optimizations are actively being developed. Solidity compiler optimizations are disabled by default, and it is unclear how many contracts in the wild actually use them. Therefore, it is unclear how well they are being tested and exercised.

Security issues due to optimization bugs have occurred in the past. A medium- to high-severity bug in the Yul optimizer was introduced in Solidity version 0.8.13 and was fixed only recently, in Solidity version 0.8.17. Another medium-severity optimization bug—one that caused memory writes in inline assembly blocks to be removed under certain conditions— was patched in Solidity 0.8.15.

A compiler audit of Solidity from November 2018 concluded that the optional optimizations may not be safe.

It is likely that there are latent bugs related to optimization and that new bugs will be introduced due to future optimizations.

**Exploit Scenario**

A latent or future bug in Solidity compiler optimizations causes a security vulnerability in the Ondo protocol contracts

**Recommendations**

Short term, measure the gas savings from optimizations and carefully weigh them against the possibility of an optimization-related bug.

Long term, monitor the development and adoption of Solidity compiler optimizations to assess their maturity.

## Risks associated with the use and design of the Huff proxy

**Description**

The design and use of the Huff proxy contract introduce risks that are not justified and may be unnecessary.

The Huff proxy is used in two places:
● The wallet factory contract
● The Nested wallet contract

The use of Huff to write the proxy is justified: to reduce the proxy’s size. However, the use of the proxy in the wallet factory is unlikely to meaningfully reduce the cost of deploying the contract.

Moreover, the proxy retrieves the implementation’s address using a resolver with dynamic dispatching:

Here, the proxy calls the implementation resolver with either “WalletFactory” or “NestedWallet”

When the implementation is constructed, the proxy appends the name of the implementation to the end of the bytecode. At runtime, the proxy then retrieves the name from the bytecode by copying its value.

All of these operations follow the same schema that Solidity’s immutable variables do, but they are done through raw opcode manipulations

This pattern is highly error-prone. Because all of these operations could be replaced with one implementation resolver per type of contract, the risks that come with using this pattern may be unnecessary.

**Recommendations**

Short term, use a Solidity proxy for the factory contract. Use one implementation resolver per type of contract. 

Long term, carefully review every design choice and always strive for simplicity. Make sure to justify every use of assembly code.

## Unclear usage of Dca.tokensOutAllocation

**Description**

The documentation of the tokensOutAllocation parameter of the Dca struct is unclear and does not match the current implementation.

The tokensOutAllocation parameter, a uint16 array, is defined in the Dca struct.

The documentation indicates that the values in tokensOutAllocation are the allocation percentage for each token

It is, however, not a required parameter because the number of tokens to be transferred are stored in the tokensOutAllocation parameter. The tokensOutAllocation parameter is used only in the updateDcaAmount function, in which the swapParams.amountIn parameter is updated.

The amounts in tokensOutAllocation are computed fractions of the amount given as a parameter. The exact percentages for each token are stored in tokensOutAllocation, which is not changeable without recreating the DCA. This introduces more complexity and gas usage and limits the number of possible values that the new amounts can take. Furthermore, it assumes that all of the input tokens in swapParams are the same.

**Recommendations**

Short term, remove tokensOutAllocation and allow the token amounts for DCA tasks to be given in absolute quantities.

Long term, carefully consider every feature added to a smart contract. Make sure the feature’s goals are clear and necessary.

## Documentation and naming conventions can be improved

**Description**

Although the documentation is extensive, there are numerous areas where the documentation is either lacking or differs from the specifications in the codebase.

**Mismatch between documentation and code**
For example, regarding the optOut functionality, the documentation states that if a position is fully matched, a lender can optOut. However, it is ambiguous if this means a lender can optOut regardless of whether the position is fully matched or partially matched:

In the validateOptOut function implementation, the function reverts if the position has not been matched at all

**Obscure naming conventions in codebase**
Some naming conventions used throughout the codebase are misleading and create confusion. For example, the didPartiallyWithdraw variable is ambiguous and implies that the current process is a partial withdrawal, when in reality the variable indicates whether a partial withdrawal has already taken place

**Incorrect variable names**
The withdrawLoanId is meant to be used only in the detach function; however, its name suggests that it should also be used to track withdrawn loans

The deposit function of the LoanLender contract uses the CREATION_TIMESTAMP to validate that the BOOK_BUILDING phase is still ongoing, as shown in figure 16.5. However, the BOOK_BUILDING phase does not actually start until the borrower executes the enableBookBuildingPhase function. This means that **BOOK_BUILDING_PERIOD_DURATION** defines the maximum duration of the period instead of the actual duration, as its name suggests.

**Duplicate variable names**
The duplicate use of variable names, even when referring to different values, creates confusion and makes reviewing the codebase difficult. The constant ONE is reused throughout the codebase and meant to indicate the decimals used for fixed-point calculations; however, it is assigned different values, such as 10**token.decimals() or 10**18. This makes it difficult to track the correct fixed point denominator.

**Recommendations**

Short term, update the documentation to correctly match the specification, and rename variables to provide clarity and remove ambiguity.

Long term, follow the following known system risks and limitations to keep documentation as close to the code as possible. Ensure that this behavior can be derived explicitly from naming conventions:
● Ensure implementation remains consistent with the specification and code comments
● Differentiate documentation between end-users and developers
● Cover all expected and unexpected behavior
● Follow similar naming conventions between code and specifications

## Documentation discrepancy in computePriceWithChangeInTau

**Description**

The formula provided in the @custom field of the NatSpec comment for the computePriceWithChangeInTau function does not match the formula implemented in the function body. The discrepancy between the documentation and implementation can cause end users to misunderstand what the function actually does.

The function body implements the following formula:

𝑃(τ − ε) = ( (𝑃(τ)/𝐾)^(√(1 − ε/τ)) ) * 𝐾 * 𝑒^((1/2)(σ^2)(√(τ)√(τ − ε) − (τ − ε)))

The documented and implemented formulas will result in different values for P(τ - ε).

**Exploit Scenario**

Alice calculates the result of her investment using formulas from the codebase’s NatSpec comments. The result of her calculations convinces her to submit transactions. When Alice completes her series of transactions, the end result differs from her expectations.

**Recommendations**

Short term, correct the formula error in the NatSpec comment for computePriceWithChangeInTau so that it matches the implementation.

Long term, thoroughly proofread the NatSpec comments throughout the codebase, especially where they describe important formulas and operations.

## Liquidity providers can withdraw total fees earned by a pool

**Description**

The syncPositionFees function is implemented incorrectly. As a result, liquidity providers can withdraw the total fees earned by a pool and drain assets from the contract reserves.

Pools earn fees from swaps, which should be distributed among the liquidity providers proportional to the liquidity they provided during the swaps. However, the Hyper contract instead distributes the total fees earned by the pool to every liquidity provider, resulting in the distribution of more tokens in fees than earned by the pool.

As shown in figure 7.1, the syncPositionFees function is used to compute the fee earned by a liquidity provider. This function multiplies the fee earned per wad of liquidity by the liquidity value, provided as an argument to the function.

The syncPositionFees function is used in the _changeLiquidity and claim functions defined in the Hyper contract. In both locations, when the syncPositionFees function is called, the value of the pool’s total liquidity is provided as the first argument, which is then multiplied by the fee earned per wad of liquidity. The value resulting from the multiplication is then added to the fee earned by the liquidity provider, which means the total fee earned by the pool is added to the fee earned by the liquidity provider.

There are multiple ways in which an attacker could use this issue to withdraw more than what they have earned in fees.

**Exploit Scenario 1**

Eve provides minimal liquidity to a pool. Eve waits for some time for some swaps to happen. She calls the claim function to withdraw the total fee earned by the pool during the period for which Alice and other users have provided liquidity.

**Exploit Scenario 2**

Eve provides minimal liquidity to a pool using 10 accounts. Eve makes some large swaps to accrue fees in the pool. She then calls the claim function from all 10 accounts to withdraw 10 times the total fee she paid for the swaps. She repeats these steps to drain the contract reserves.

**Recommendations**

Short term, revise the relevant code so that the value of the sum of a liquidity provider’s liquidity (freeLiquidity summed with stakedLiquidity) is passed as an argument to the syncPositionFees function instead of the entire pool’s liquidity.

Long term, take the following actions:

● In all functions, document their arguments, their meanings, and their usage.
● Add unit test cases that check all happy and unhappy paths.
● Identify additional system invariants and implement Echidna to capture bugs related to them.

## Asset token price deviates from the price curve of the pool

**Description**

The asset token price deviates from the price curve of the pool with each swap operation because of the price adjustment performed in the swap function.

On each swap, the _swapExactIn function computes new virtual reserves of a pool based on the user’s input and then computes the new price of the asset token based on the pool’s new virtual reserves. As shown in figure 8.1, a factor of 10 11 wei is added to the calculated price.

This adjusted price is then used to calculate the output amount in the next swap operation. The price of the asset token increases by the factor of 10 11 wei with each swap operation, which causes the price of the asset token to deviate from the price curve of the pool.

This adjusted price is also used to calculate the pool’s new virtual reserves on subsequent swaps, which means that the virtual reserves of the pool will be different from those used to calculate this price. This creates a difference between the contract’s balance of pool tokens and the pool’s virtual reserves; this difference increases with every swap operation, causing assets to become stuck in the contract.

**Recommendations**

Short term, investigate the maximum amount of deviation that can occur between the asset token price and the price curve of the pool to ensure that the error fits within safe bounds.

Long term, execute thorough economic analysis on the implications of any update to system variables. This should include maximum error calculations for all variables.

## New pair creation can overwrite existing pairs

**Description**

An overflow of the maximum value of uint24 could occur in the _createPair() function, which can be used to overwrite existing pairs.

As shown in figure 9.1, the _createPair() function uses an unchecked block to compute and cast the value of getPairNonce to assign the value of pairId. This pairId is used as a key in the pairs mapping to store information related to a specific pair.

According to the Solidity documentation, values that undergo explicit type conversion are truncated if the new type cannot hold all of the bits required to represent the new value. Here, the type of getPairNonce is converted from uint256 to uint24, so if the new value overflows the maximum uint24 value, the higher-order bits of getPairNonce will be truncated to an unexpected value. This means that existing pools will be overwritten by every new pair creation operation, resulting in an inconsistent contract state with the following issues:

● Two sets of assets will store the same pairId in the getPairId mapping.
● The value of HyperPair will be overwritten in the pairs mapping.
● All of the previous pools created for the pair will still hold the previous value of HyperPair.

This overflow limit may not seem feasible to reach because the maximum value of the uint24 type is 16,777,215. It would take a lot of time and cost a lot of gas to create so many pairs. However, because of the low-cost nature and higher transaction throughput of the L2 networks, a malicious user could exploit this issue to conduct a DoS attack on the protocol with insignificant financial loss.

We also found that a pool existence check is implemented in _createPool function, but an inline comment indicates Primitive’s plans to remove this check. We recommend keeping this check to prevent similar poolId overflow and overwriting issues.

**Exploit Scenario**

Eve deploys numerous fake ERC-20 token contracts. She then uses these ERC-20 tokens to create 16,777,215 bogus pairs in the Hyper contract. Now, every new pair overwrites existing pairs, making this instance of the protocol unusable.

**Recommendations**

Short term, make the following changes:

1. Add a check of the pairId in _createPair() to ensure that it has not already been used to prevent existing pairs from being overwritten.
2. Increase the upper bound of the value of pairId by changing its type.

Long term, carefully review the codebase for explicit type conversions. Document scenarios in which these explicit type conversions can result in an overflow or underflow of the result, and use Echidna to test for these scenarios throughout the codebase.

## Error in Invariant.getX

**Description**

The getX function in the Invariant contract implements a formula that does not align with the formula specified in the white paper.

The body of the function matches the formula described in the NatSpec comment of the function. However, the formula itself is derived incorrectly. The actual quantity to be used inside the parentheses should be (y - k) instead of (y + k).

The formula is derived by rearranging the terms in the following:

y = KΦ(Φ−1 (1 − x) − σ √ τ) + k

By subtracting k from both sides of the equation, it becomes clear that the formula should read (y - k).

The actual impact of this error in the current implementation is low because the function is only ever called with the invariant k = 0.

**Exploit Scenario**

In a future release of the protocol, Primitive decides to use the function with the parameter k != 0. This breaks the desired invariant after swaps occur.

**Recommendations**

Short term, correct the getX function’s code and update the formula in the function’s NatSpec comment.

Long term, keep track of the derivations of formulas used throughout the codebase, and add fuzz tests that verify the properties of and assumptions about the functions that implement them.

## Pools with overflowing maturity dates can be created

**Description**

Users could create pools with maturity date timestamps (which are of type uint32) that are too close to when uint32 timestamps overflow; pools with overflowing timestamps would become unusable.

Maturity dates are currently limited to five years in the future, and the date when uint32 timestamps will overflow is September 25, 2104. The maturity date is not validated on pool creation, so pools that will be unusable when the year 2104 approaches could be created.

The maturity date is also not validated in the checkParameters() function of the HyperLib contract, which could allow an attacker to set the timestamp parameter of the pool to an overflowing timestamp.

**Exploit Scenario**

In a future version of the protocol, Primitive removes the five-year limit on pool maturity dates. Alice, the controller of a pool, decides to set the maturity date past the year 2104 and is able to trap all of the funds of the pool’s liquidity providers.

**Recommendations**

Short term, add a check to the checkParameters() function in HyperLib to ensure that pools’ maturity dates will not overflow.

Long term, thoroughly document all of the assumptions that are made on the codebase’s variables. Where types are limited in size, use modular arithmetic or a larger data type to handle any issues that could occur.

## Attackers can sandwich changeParameters calls to steal funds

**Description**

The changeParameters function does not adjust the reserves according to the new parameters, which results in a discrepancy between the virtual reserves of a pool and the reserves of the tokens in the Hyper contract.

An attacker can create a new controlled pool with the token they want to steal and a fake token. They can then use a sequence of operations—allocate, changeParameters, and unallocate—to steal tokens from the shared reserves of the Hyper contract. In the worst-case scenario, an attacker can drain all of the funds from the Hyper contract.

The Hyper contract does not store the reserves of tokens in a pool. The virtual reserves of the pool are computed with the last price of the given asset token and the curve parameters. Below, we discuss the impact that a change in parameters could have on various operations:

** Allocating and Unallocating**
A user can add liquidity to or remove liquidity from a pool using the allocate and unallocate functions. Both of these functions use the getLiquidityDeltas function to compute the amount of the asset token and the amount of the quote token required to change the liquidity by the desired amount. The getLiquidityDeltas function calls the getAmountsWad function to compute the amount of reserves required for adding one unit of liquidity to the pool. The getAmountsWad function uses the last price of the asset token (self.lastPrice), strike price, time to maturity, and implied volatility to compute the amount of reserves per liquidity.

When a controller calls the changeParameters function, the function updates the pool parameter values stored in the Hyper contract to the provided values. These new values are then used in the next execution of the getAmountsWad function along with the value of pool.lastPrice, which was computed with the previous pool parameters. If getAmountsWad uses the previous pool.lastPrice value with new pool parameters, it will return new values for the reserves of the pool; however, the Hyper contract’s token balances will match those computed with the previous pool parameters.

When a user calls the unallocate function after a pool’s parameters have been updated, the new computed reserve amounts are used to transfer tokens to the user. Because the Hyper contract uses shared reserves of tokens for all of the pools, a user can still withdraw the tokens, allowing them to steal tokens from other pools.

**Swapping**
The swap function computes the price of the asset token using the _computeSyncedPrice function. This function uses the pool parameters to compute the price of the asset token. It uses pool.lastPrice as the current price of the asset token, as shown in figure 15.2. It then calls the computePriceChangeWithTime function with the pool parameters and the time elapsed since the last swap operation.

The computePriceChangeWithTime function computes the strike price of the pool and then calls the Price.computeWithChangeInTau function to get the current price of the asset token.

The computePriceChangeWithTime function computes the strike price of the pool and then calls the Price.computeWithChangeInTau function to get the current price of the asset token.

If a user swaps tokens after the controller has updated the curve parameters, then the wrong price computed by the swap function will result in unexpected behavior. This issue can be used by the controller of the pool to swap at a discounted rate.

There are multiple ways in which an attacker could use this issue to steal funds from the Hyper contract.

**Exploit Scenario 1**

Eve creates a new controlled pool with WETH as an asset token and a fake token as a quote token. Eve allocates 1e18 wad of liquidity in the new pool by depositing X amount of WETH and Y amount of the fake token. Eve then doubles the strike price of the pool by calling changeParameters(). This change in the strike price changes the virtual reserves of the pool; specifically, it increases the number of asset tokens and decreases the number of quote tokens required to allocate 1e18 wad of liquidity. Eve then unallocates 1e18 wad liquidity and withdraws X1 amount of WETH and Y1 amount of the fake token. The value of X1 is higher than X because of the change in the strike price. This allows Eve to withdraw more WETH than she deposited.

**Exploit Scenario 2**

Eve creates a controlled pool for two popular tokens. Other users add liquidity to the pool. Eve then changes the pool’s parameters to change the strike price to a value in her favor and executes a large swap to benefit from the liquidity added to the pool. The other users see Eve’s actions as arbitrage and lose the value of their provided liquidity. Eve has effectively swapped tokens at a discounted rate and stolen funds from the pool’s liquidity providers.

**Recommendations**

Short term, modify the changeParameters function so that it computes new token reserve amounts for pools based on updated pool parameters and transfers tokens to or from the controller to align the Hyper contract reserves with the new pool reserves.

Long term, carefully review the codebase to find instances in which the assumptions used in formulas become invalid because of user actions; resolve issues arising from the use of invalid formulas. Add fuzz tests to capture such instances in the codebase.

## Functions that round by adding 1 result in unexpected behavior

**Description**

Unit conversion functions in the Assembly contract add 1 to the results of their division operations to round them up, which results in unexpected rounding effects.

The scaleFromWadUp() function is used to convert the input for swap operations from a wad unit to a token decimal unit. The function always adds 1 to the result of the division operation, even if the input amount would be a whole number in token decimals. As a result, the user will transfer more tokens than expected.

The scaleFromWadUpSigned() also adds 1 to the result of the division operation to round up the return value.

**Recommendations**

Short term, use the formula ((a - 1)/b) + 1 in the scaleFromWadUp() and scaleFromWadUpSigned() functions to compute the rounded up result of the division operation a/b.

Long term, review the entire codebase for functions that round to ensure that they do not add 1 unconditionally to results. Add fuzzing test cases to find edge cases that could cause unexpected rounding issues.

**References**
● Number Logic

## Solidity compiler optimizations can be problematic

**Description** 

he Hyper contracts have enabled optional compiler optimizations in Solidity.

There have been several optimization bugs with security implications. Moreover, optimizations are actively being developed. Solidity compiler optimizations are disabled by default, and it is unclear how many contracts in the wild actually use them. Therefore, it is unclear how well they are being tested and exercised.

High-severity security issues due to optimization bugs have occurred in the past. A high-severity bug in the emscripten-generated solc-js compiler used by Truffle and Remix persisted until late 2018. The fix for this bug was not reported in the Solidity CHANGELOG. Another high-severity optimization bug resulting in incorrect bit shift results was patched in Solidity 0.5.6. More recently, another bug due to the incorrect caching of keccak256 was reported.

A compiler audit of Solidity from November 2018 concluded that the optional optimizations may not be safe. 

It is likely that there are latent bugs related to optimization and that new bugs will be introduced due to future optimizations.

**Exploit Scenario**

A latent or future bug in Solidity compiler optimizations—or in the Emscripten transpilation to solc-js— opens up a security vulnerability in the Solstat contracts.

**Recommendations**

Short term, measure the gas savings from optimizations and carefully weigh them against the possibility of an optimization-related bug.

Long term, monitor the development and adoption of Solidity compiler optimizations to assess their maturity.

## Incorrect implementation of edge cases in getY function

**Description**

The getY function in the Invariant contract deviates from the intended behavior when the value of R_x is equivalent to WAD and 0.

The first if statement seems to be checking for the ppf(1) case, evinced by the comparison of R_x == WAD, with WAD representing one unit. The function should return the invariant in this case, but it returns the stk value summed with the invariant.

Similarly, when R_x is 0, the return value should be stk summed with the invariant, but it is simply the invariant.

Therefore, the return values for the two branches are incorrect.

**Exploit Scenario** 

Alice attempts to execute a swap, for which the R_x value is equivalent to 0. The function returns the invariant rather than stk summed with the invariant. This results in further miscalculations in the swaps.

**Recommendations**

Short term, switch the return value statements in the two affected branches of the getY function: if R_x is WAD, the function should return the invariant, and if R_x is 0, the function should return stk summed with the invariant. 

Long term, thoroughly document all of the expected edge cases of inputs and check that these edge cases are handled.

## Lack of proper bound handling for solstat functions

**Description**

The use of unchecked assembly across the system combined with a lack of data validation means that obscure bugs that are difficult to track down may be prevalent in the system.

One example of unchecked assembly that could result in bugs is the getY function. Due to assembly rounding issues in the codebase, the inverse of the function’s two variables does not hold. This function uses targeted functions in the Gaussian code. We wrote a fuzz test and ran it on the getY function to ensure that the return values are monotonically decreasing. However, the fuzzing campaign found cases in which the getY function returns a lower value than it should:

The solstat Gaussian contract contains the cumulative distribution function (cdf), which relies on the erfc function. The erfc function, however, has multiple issues, indicated by its many breaking invariants; for example, it is not monotonic, it returns hard-coded values outside of its input domain, it has inconsistent rounding directions, and it is missing overflow protection (further described below). This means that the cdf function’s assumption that it always returns a maximum error of 1.2e-7 may not hold under all conditions:

The erfc function (and related functions) are used throughout the Hyper contract to compute the contract’s reserves, prices, and the invariants. This function contains a few issues, described below the figure.

**Lack of Overflow Checks**
The erfc function does not contain overflow checks. In the above assembly block, the first multiplication operation does not check for overflow. Operations performed in an assembly block use unchecked arithmetic by default. If z, the absolute value of the input, is larger than ⌈type(int256).max / 1e18⌉ (rounded up), the multiplication operation will result in an overflow.

**Use of sdiv Instead of div**
Additionally, the erfc function uses the sdiv function instead of the div function on the result of the multiplication operation. The div function should be used instead because z and the product are positive. Even if the result of the previous multiplication operation does not overflow, if the result is larger than type(int256).max, then it will be incorrectly interpreted as a negative number due to the use of sdiv.

Because of these issues, the output values could lie well beyond the intended output domain of the function, [0, 2]. For example, erfc(x) ≈ 1e57 is a possible output value.

Other functions—and those that rely on erfc, such as pdf, ierfc, cdf, ppf, getX, and getY—are similarly affected and could produce unexpected results. Some of these issues are further outlined in appendix E.

Due to the high complexity and use of the function throughout this codebase, the exact implications of an incorrect bound on the function are unclear. We specify further areas that require investigation in appendix C; however, Primitive should conduct additional analysis on the precision loss and specificity of solstat functions

**Exploit Scenario**

An attacker sees that under a certain swap configuration, the output amount in Hyper’s swap function will result in a significant advantage for the attacker.

**Recommendations**

Short term, rewrite all of the affected code in high-level Solidity with native overflow protection enabled.

Long term, set up sufficient invariant tests using Echidna that can detect these issues in the code. For all functions, perform thorough analysis on the valid input range, document all assumptions, and ensure that all functions revert if the assumptions on the inputs do not hold.

## Attackers can steal funds by swapping in both directions

**Description**

Unexpected behavior in the swap function allows users to profit when executing a swap in one direction and then executing the same swap in the other direction.

In a fuzz test, we identified a case in which an attacker is able to create a pool with a certain configuration that allows them to swap in 1 wei of the asset token and then to swap back out a high number of asset tokens. This would allow the attacker to drain the pool. This issue was found toward the end of the audit, so we were unable to locate the root cause.

**Recommendations**

Short term, analyze the swap function to identify the root cause of the vulnerability. This issue still persists after fixing overflow issues and unit conversions by replacing assembly code with high-level code.

Long term, add fuzz test cases to find edge cases causing issues with unexpected rounding behavior.

## Solidity compiler optimizations can be problematic

**Description**

Spool V2 has enabled optional compiler optimizations in Solidity.

There have been several optimization bugs with security implications. Moreover, optimizations are actively being developed . Solidity compiler optimizations are disabled by default, and it is unclear how many contracts in the wild actually use them. Therefore, it is unclear how well they are being tested and exercised.

High-severity security issues due to optimization bugs have occurred in the past . A high-severity bug in the emscripten -generated solc-js compiler used by Truffle and Remix persisted until late 2018. The fix for this bug was not reported in the Solidity CHANGELOG. Another high-severity optimization bug resulting in incorrect bit shift results was patched in Solidity 0.5.6 . More recently, another bug due to the incorrect caching of keccak256 was reported.

A compiler audit of Solidity from November 2018 concluded that the optional optimizations may not be safe.

It is likely that there are latent bugs related to optimization and that new bugs will be introduced due to future optimizations.

**Exploit Scenario**

A latent or future bug in Solidity compiler optimizations—or in the Emscripten transpilation to solc-js —causes a security vulnerability in the Spool V2 contracts.

**Recommendations**

Short term, measure the gas savings from optimizations and carefully weigh them against the possibility of an optimization-related bug.

Long term, monitor the development and adoption of Solidity compiler optimizations to assess their maturity.

## Strategy APYs are never updated

**Description**

The _updateDhwYieldAndApy function is never called. As a result, each strategy’s APY will constantly be set to 0

A strategy’s APY is one of the parameters used by an allocator provider to decide where to allocate the assets of a smart vault. If a strategy’s APY is 0 , the LinearAllocationProvider and ExponentialAllocationProvider contracts will both revert when calculateAllocation is called due to a division-by-zero error.

When a vault is created, the code in figure 8.2 is executed. For vaults whose strategyAllocation variable is set to 0 , which means the value will be calculated by the smart contract, and whose allocationProvide r variable is set to the LinearAllocationProvider or ExponentialAllocationProvider contract, the creation transaction will revert due to a division-by-zero error. Transactions for creating vaults with a nonzero strategyAllocation and with the same allocationProvider values mentioned above will succeed; however, the fund reallocation operation will revert because the _updateDhwYieldAndApy function is never called, causing the strategies’ APYs to be set to 0 , in turn causing the same division-by-zero error. Refer to finding TOB-SPL-7 , which is related to this issue; even if that finding is fixed, incorrect results would still occur because of the missing _updateDhwYieldAndApy calls.

**Exploit Scenario**

Bob tries to deploy a smart vault with strategyAllocation set to 0 and allocationProvide r set to LinearAllocationProvider . The transaction unexpectedly fails.

**Recommendations**

Short term, add calls to _updateDhwYieldAndApy where appropriate.

Long term, improve the system’s unit and integration tests to ensure that the basic operations work as expected.

## Incorrect bookkeeping of assets deposited into smart vaults

**Description**

Assets deposited by users into smart vaults are incorrectly tracked. As a result, assets deposited into a smart vault’s strategies when the flushSmartVault function is invoked correspond to the last deposit instead of the sum of all deposits into the strategies.

When depositing assets into a smart vault, users can decide whether to invoke the flushSmartVault function. A smart vault flush is a synchronization process that makes deposited funds available to be deployed into the strategies and makes withdrawn funds available to be withdrawn from the strategies. However, the internal bookkeeping of deposits keeps track of only the last deposit of the current flush cycle instead of the sum of all deposits (figure 9.1).

The _vaultDeposits variable is then used to calculate the asset distribution in the flushSmartVault function.

Lastly, the _strategyRegistry.addDeposits function is called with the computed distribution, which adds the amounts to deploy in the next doHardWork function call in the _assetsDeposited variable (figure 9.3).

The next time the doHardWork function is called, it will transfer the equivalent of the last deposit’s amount instead of the sum of all deposits from the master wallet to the assigned strategy (figure 9.4).

**Exploit Scenario**

Bob deploys a smart vault. One hundred deposits are made before a smart vault flush is invoked, but only the last deposit’s assets are deployed to the underlying strategies, severely impacting the smart vault’s performance.

**Recommendations**

Short term, modify the depositAssets function so that it correctly tracks all deposits within a flush cycle, rather than just the last deposit.

Long term, improve the system’s unit and integration tests: test a smart vault with a single strategy and multiple strategies to ensure that smart vaults behave correctly when funds are deposited and deployed to the underlying strategies.

## GuardManager does not account for all possible types when encoding guard arguments

**Description**

While encoding arguments for guard contracts, the GuardManager contract assumes that all static types are encoded to 32 bytes. This assumption does not hold for fixed-size static arrays and structs with only static type members. As a result, guard contracts could receive incorrect arguments, leading to unintended behavior.

The GuardManager._encodeFunctionCall function manually encodes arguments to call guard contracts (figure 11.1).

The function calculates the offset for dynamic type arguments assuming that every parameter, static or dynamic, takes exactly 32 bytes. However, fixed-length static type arrays and structs with only static type members are considered static. All static type values are encoded in-place, and static arrays and static structs could take more than 32 bytes.

As a result, the calculated offset for the start of dynamic type arguments could be wrong, which would cause incorrect values for these arguments to be set, resulting in unintended behavior. For example, the guard could approve invalid user actions and reject valid user actions or revert every call.

**Exploit Scenario**

Bob deploys a smart vault and creates a guard contract that takes the custom value of a f ixed-length static array type. The guard contract uses RequestContext assets. Bob correctly creates the guard definition in GuardManager , but the GuardManager._encodeFunctionCall function incorrectly encodes the arguments. The guard contract fails to decode the arguments and always reverts the execution.

**Recommendations**

Short term, modify the GuardManager._encodeFunctionCall function so that it considers the encoding length of the individual parameters and calculates the offsets correctly.

Long term, avoid implementing low-level manipulations. If such implementations are unavoidable, carefully review the Solidity documentation before implementing them to ensure that they are implemented correctly.

## Use of encoded values in guard contract comparisons could lead to opposite results

**Description**

The GuardManager contract compares the return value of a guard contract to an expected value. However, the contract uses encoded versions of these values in the comparison, which could lead to incorrect results for signed values with numerical comparison operators.

The GuardManager contract calls the guard contract and validates the return value using the GuardManager._checkResult function (figure 12.1).

When a smart vault creator defines a guard using the GuardManager.setGuards function, they define a comparison operator and the expected value, which the GuardManager contract uses to compare with the return value of the guard contract.

The comparison is performed on the first 32 bytes of the ABI-encoded return value and the expected value, which will cause issues depending on the return value type.

First, the numerical comparison operators ( < , > , <= , >= ) are not well defined for bytes32 ; therefore, the contract treats encoded values with padding as uint256 values before comparing them. This way of comparing values gives incorrect results for negative values of the int type. The Solidity documentation includes the following description about the encoding of int type values:

Because negative values are padded with 0xff and positive values with 0x00 , the encoded negative values will be considered greater than the encoded positive values. As a result, the result of the comparison will be the opposite of the expected result.

Second, only the first 32 bytes of the return value are considered for comparison. This will lead to inaccurate results for return types that use more than 32 bytes to encode the value.

**Exploit Scenario**

Bob deploys a smart vault and intends to allow only users who own B NFTs to use it. B NFTs are implemented using ERC-1155. Bob uses the B contract as a guard with the comparison operator > and an expected value of 0 .

Bob calls the function B.balanceOfBatch to fetch the NFT balance of the user. B.balanceOfBatch returns uint256[] . The first 32 bytes of the return data contain the offset into the return data, which is always nonzero. The comparison passes for every user regardless of whether they own a B NFT. As a result, every user can use Bob’s smart vault.

**Recommendations**

Short term, restrict the return value of a guard contract to a Boolean value. If that is not possible, document the limitations and risks surrounding the guard contracts. Additionally, consider manually checking new action guards with respect to these limitations.

Long term, avoid implementing low-level manipulations. If such implementations are unavoidable, carefully review the Solidity documentation before implementing them to ensure that they are implemented correctly.

## Incorrect use of exchangeRates in doHardWork

**Description**

The StrategyRegistry contract’s doHardWork function fetches the exchangeRates for all of the tokens involved in the “do hard work” process, and then it iterates over the strategies and saves the exchangeRates values for the current strategy’s tokens in the assetGroupExchangeRates variable; however, when doHardWork is called for a strategy, the exchangeRates variable rather than the assetGroupExchangeRates variable is passed, resulting in the use of incorrect exchange rates.

The exchangeRates values are used by a strategy’s doHardWork function to calculate how many assets in USD value are to be deposited and how many in USD value are currently deposited in the strategy. As a consequence of using exchangeRates rather than assetGroupExchangeRates , the contract will return incorrect values.

Additionally, the _exchangeRates variable is returned by the strategyAtIndexBatch function, which is used when simulating deposits.

**Exploit Scenario**

Bob deploys a smart vault, and users start depositing into it. However, the first time doHardWork is called, they notice that the deposited assets and the reported USD value deposited into the strategies are incorrect. They panic and start withdrawing all of the funds.

**Recommendations**

Short term, replace exchangeRates with assetGroupExchangeRates in the relevant areas of doHardWork and where it sets the _exchangeRates variable.

Long term, improve the system’s unit and integration tests to verify that the deposited value in a strategy is the expected amount. Additionally, when reviewing the code, look for local variables that are set but then never used; this is a warning sign that problems may arise.

## LinearAllocationProvider could return an incorrect result

**Description**

The LinearAllocationProvider contract returns an incorrect result when the given smart vault has a riskTolerance value of -8 due to an incorrect literal value in the riskArray variable.

The riskArray ’s third element is incorrect; this affects the computed allocation for smart vaults that have a riskTolerance value of -8 because the riskt variable would be 2 , which is later used as index for the riskArray .

The subexpression risk * riskArray[uint8(rikst)] is incorrect by a factor of 10.

**Exploit Scenario**

Bob deploys a smart vault with a riskTolerance value of -8 and an empty strategyAllocation value. The allocation between the strategies is computed on the spot using the LinearAllocationProvider contract, but the allocation is wrong.

Recommendations

Short term, replace 900000 with 90000 in the calculateAllocation function.

Long term, improve the system’s unit and integration tests to catch issues such as this. Document the use and meaning of constants such as the values in riskArray . This will make it more likely that the Spool team will find these types of mistakes.

## Incorrect formula used for adding/subtracting two yields

**Description**

The doHardWork function adds two yields with different base values to compute the given strategy’s total yield, which results in the collection of fewer ecosystem fees and treasury fees.

It is incorrect to add two yields that have different base values. The correct formula to compute the total yield from two consecutive yields Y1 and Y2 is Y1 + Y2 + (Y1*Y2)

The doHardWork function in the Strategy contract adds the protocol yield and the rewards yield to calculate the given strategy’s total yield. The protocol yield percentage is calculated with the base value of the strategy’s total assets at the start of the current “do hard work” cycle, while the rewards yield percentage is calculated with the base value of the total assets currently owned by the strategy.

Therefore, the total yield of the strategy is computed as less than its actual yield, and the use of this value to compute fees results in the collection of fewer fees for the platform’s governance system.

Same issue also affects the computation of the total yield of a strategy on every “do hard work” cycle:

This value of the total yield of a strategy is used to calculate the management fees for a given smart vault, which results in fewer fees paid to the smart vault owner.

**Exploit Scenario**

The Spool team deploys the system. Alice deposits 1,000 tokens into a vault, which mints 1,000 strategy share tokens for the vault. On the next “do hard work” execution, the tokens earn 8% yield and 30 reward tokens from the protocol. The 30 reward tokens are then exchanged for 20 deposit tokens. At this point, the total tokens earned by the strategy are 100 and the total yield is 10%. However, the doHardWork function computes the total yield as 9.85%, which is incorrect, resulting in fewer fees collected for the platform.

**Recommendations**

Short term, use the correct formula to calculate a given strategy’s total yield in both the Strategy contract and the StrategyRegistry contract.

Note that the syncDepositsSimulate function subtracts a strategy’s total yield at different “do hard work” indexes in DepositManager.sol#L322–L326 to compute the difference between the strategy’s yields between two “do hard work” cycles. After fixing this issue, this function’s computation will be incorrect.

Long term, review the entire codebase to find all of the mathematical formulas used. Document these formulas, their assumptions, and their derivations to avoid the use of incorrect formulas.


## Smart vaults with re-registered strategies will not be usable

**Description**

The StrategyRegistry contract does not clear the state related to a strategy when removing it. As a result, if the removed strategy is registered again, the StrategyRegistry contract will still contain the strategy’s previous state, resulting in a temporary DoS of the smart vaults using it. 

The StrategyRegistry.registerStrategy function is used to register a strategy and to initialize the state related to it (figure 17.1). StrategyRegistry tracks the state of the strategies by using their address.

The StrategyRegistry._removeStrategy function is used to remove a strategy by revoking its ROLE_STRATEGY role.

While removing a strategy, StrategyRegistry contract does not remove the state related to that strategy. As a result, when that strategy is registered again, StrategyRegistry will contain values from the previous period. This could make the smart vaults using the strategy unusable or cause the unintended transfer of assets between other strategies and this strategy.

Exploit Scenario Strategy S is registered. StrategyRegistry._currentIndex[S] is equal to 1 . Alice creates a smart vault X that uses strategy S. Bob deposits 1 million WETH into smart vault X. StrategyRegistry._assetsDeposited[S][1][WETH] is equal to 1 million WETH. The doHardWork function is called for strategy S. WETH is transferred from the master wallet to strategy S and is deposited into the protocol.

A Spool system admin removes strategy S upon hearing that the protocol is being exploited. However, the admin realizes that the protocol is not being exploited and re-registers strategy S. StrategyRegistry._currentIndex[S] is set to 1 . StrategyRegistry._assetsDeposited[S][1][WETH] is not set to zero and is still equal to 1 million WETH.

Alice creates a new vault with strategy S. When doHardWork is called for strategy S, StrategyRegistry tries to transfer 1 million WETH to the strategy. The master wallet does not have those assets, so doHardWork fails for strategy S. The smart vault becomes unusable.

**Recommendations**

Short term, modify the StrategyRegistry._removeStrategy function so that it clears states related to removed strategies if re-registering strategies is an intended use case. If this is not an intended use case, modify the StrategyRegistry.registerStrategy function so that it verifies that newly registered strategies have not been previously registered.

Long term, properly document all intended use cases of the system and implement comprehensive tests to ensure that the system behaves as expected.

## Incorrect handling of partially burned NFTs results in incorrect SVT balance calculation

**Description**

The SmartVault._afterTokenTransfer function removes the given NFT ID from the SmartVault._activeUserNFTIds array even if only a fraction of it is burned. As a result, the SmartVaultManager.getUserSVTBalance function, which uses SmartVault._activeUserNFTIds , will show less than the given user’s actual balance. 

SmartVault._afterTokenTransfer is executed after every token transfer (figure 18.1).

It removes the burned NFT from _activeUserNFTIds . However, it does not consider the amount of the NFT that was burned. As a result, NFTs that are not completely burned will not be considered active by the vault. 

SmartVaultManager.getUserSVTBalance uses SmartVault._activeUserNFTIds to calculate a given user’s SVT balance (figure 18.2).

Because partial NFTs are not present in SmartVault._activeUserNFTIds , the calculated balance will be less than the user’s actual balance. The front end using getUserSVTBalance will show incorrect balances to users.

**Exploit Scenario**

Alice deposits assets into a smart vault and receives a D-NFT. Alice's assets are deposited to the protocols after doHardWork is called. Alice claims SVTs by burning a fraction of her D-NFT. The smart vault removes the D-NFT from _activeUserNFTIds . Alice checks her SVT balance and panics when she sees less than what she expected. She withdraws all of her assets from the system.

**Recommendations**

Short term, add a check to the _afterTokenTransfer function so that it checks the balance of the NFT that is burned and removes the NFT from _activeUserNFTIds only when the NFT is burned completely.

Long term, improve the system’s unit and integration tests to extensively test view functions.

## Reward configuration not initialized properly when reward is zero

**Description**

The RewardManager.addToken function, which adds a new reward token for the given smart vault, does not initialize all configuration variables when the initial reward is zero. As a result, all calls to the RewardManager.extendRewardEmission function will fail, and rewards cannot be added for that vault. 

RewardManager.addToken adds a new reward token for the given smart vault. The reward tokens for a smart vault are tracked using the RewardManager.rewardConfiguration function. The tokenAdded value of the configuration is used to check whether the token has already been added for the vault (figure 22.1).

However, RewardManager.addToken does not update config.tokenAdded , and the _extendRewardEmission function, which updates config.tokenAdded , is called only when the reward is greater than zero. 

RewardManager.extendRewardEmission is the only entry point to add rewards for a vault. It checks whether token has been previously added by verifying that tokenAdded is greater than zero (figure 22.2).

Because tokenAdded is not initialized when the initial rewards are zero, the vault admin cannot add the rewards for the vault in that token.

The impact of this issue is lower because the vault admin can use the RewardManager.removeReward function to remove the token and add it again with a nonzero initial reward. Note that the vault admin can only remove the token without blacklisting it because the config.periodFinish value is also not initialized when the initial reward is zero.

**Exploit Scenario**

Alice is the admin of a smart vault. She adds a reward token for her smart vault with the initial reward set to zero. Alice tries to add rewards using extendRewardEmission , and the transaction fails. She cannot add rewards for her smart vault. She has to remove the token and re-add it with a nonzero initial reward.

**Recommendations**

Short term, use a separate Boolean variable to track whether a token has been added for a smart vault, and have RewardManager.addToken initialize that variable. 

Long term, improve the system’s unit tests to cover all execution paths.

## Missing function for removing reward tokens from the blacklist

**Description**

Finding ID: TOB-SPL-23 A Spool admin can blacklist a reward token for a smart vault through the RewardManager contract, but they cannot remove it from the blacklist. As a result, a reward token cannot be used again once it is blacklisted. The 

RewardManager.forceRemoveReward function blacklists the given reward token by updating the RewardManager.tokenBlacklist array (figure 23.1). Blacklisted tokens cannot be used as rewards.

However, RewardManager does not have a function to remove tokens from the blacklist. As a result, if the Spool admin accidentally blacklists a token, then the smart vault admin will never be able to use that token to send rewards

**Exploit Scenario** 

Alice is the admin of a smart vault. She adds WETH and token A as rewards. The value of token A declines rapidly, so a Spool admin decides to blacklist the token for Alice’s vault. The Spool admin accidentally supplies the WETH address in the call to forceRemoveReward . As a result, WETH is blacklisted, and Alice cannot send rewards in WETH.

**Recommendations**

Short term, add a function with the proper access controls to remove tokens from the blacklist.

Long term, improve the system’s unit tests to cover all execution paths.

## Risk of unclaimed shares due to loss of precision in reallocation operations

**Description**

The ReallocationLib.calculateReallocation function releases strategy shares and calculates their USD value. The USD value is later converted into strategy shares in the ReallocationLib.doReallocation function. Because the conversion operations always round down, the number of shares calculated in doReallocation will be less than the shares released in calculateReallocation . As a result, some shares released in calculateReallocation will be unclaimed, as ReallocationLib distributes only the shares computed in doReallocation .

ReallocationLib.calculateAllocation calculates the USD value that needs to be withdrawn from each of the strategies used by smart vaults (figure 24.1). The smart vaults release the shares equivalent to the calculated USD value.

The ReallocationLib.buildReallocationTable function calculates the reallocationTable value. The reallocationTable[i][j][0] value represents the USD amount that should move from strategy i to strategy j (figure 24.2). These USD amounts are calculated using the USD values of the released shares computed in ReallocationLib.calculateReallocation (represented by reallocation[0][i] in figure 24.1).

ReallocationLib.doReallocation calculates the total USD amount that should be withdrawn from a strategy (figure 24.3). This total USD amount is exactly equal to the sum of the USD values needed to be withdrawn from the strategy for each of the smart vaults. The doReallocation function converts the total USD value to the equivalent number of strategy shares. The ReallocationLib library withdraws this exact number of shares from the strategy and distributes them to other strategies that require deposits of these shares.

Theoretically, the shares calculated for a strategy should be equal to the shares released by all of the smart vaults for that strategy. However, there is a loss of precision in both the calculateReallocation function’s calculation of the USD value of released shares and the doReallocation function’s conversion of the combined USD value to strategy shares. As a result, the number of shares released by all of the smart vaults will be less than the shares calculated in calculateReallocation . Because the ReallocationLib library only distributes these calculated shares, there will be some unclaimed strategy shares as dust.

It is important to note that the rounding error could be greater than one in the context of multiple smart vaults. Additionally, the error could be even greater if the conversion results were rounded in the opposite direction: in that case, if the calculated shares were greater than the released shares, the reallocation would fail when burn and claim operations are executed.

**Recommendations**

Short term, modify the code so that it stores the number of shares released in calculateReallocation , and implement dustless calculations to build the reallocationTable value with the share amounts and the USD amounts. Have doReallocation use this reallocationTable value to calculate the value of sharesToDistribute.

Long term, use Echidna to test system and mathematical invariants.

## Curve3CoinPoolAdapter’s _addLiquidity reverts due to incorrect amounts deposited


**Description**

The _addLiquidity function loops through the amounts array but uses an additional element to keep track of whether deposits need to be made in the Strategy.doHardWork function. As a result, _addLiquidity overwrites the number of tokens to send for the first asset, causing far fewer tokens to be deposited than expected, thus causing the transaction to revert due to the slippage check.

The last element in the doHardWork function’s assetsToDeposit array keeps track of the deposits to be made and is incremented by one on each iteration of assets in assetGroup if that asset has tokens to deposit. This variable is then passed to the _depositToProtocol function and then, for strategies that use the Curve3CoinPoolAdapter , is passed to _addLiquidity in the amounts parameter. When _addLiquidity iterates over the last element in the amounts array, the assetMapping().get(i) function will return 0 because i in assetMapping is uninitialized. This return value will overwrite the number of tokens to deposit for the first asset with a strictly smaller amount.

**Exploit Scenario**

The doHardWork function is called for a smart vault that uses the ConvexAlusdStrategy strategy; however, the subsequent call to _addLiquidity reverts due to the incorrect number of assets that it is trying to deposit. The smart vault is unusable.

**Recommendations**

Short term, have _addLiquidity loop the amounts array for N_COINS time instead of its length.

Long term, refactor the Strategy.doHardWork function so that it does not use an additional element in the assetsToDeposit array to keep track of whether deposits need to be made. Instead, use a separate Boolean variable. The current pattern is too error-prone.

## Missing whenNotPaused modifier

**Description**

Finding ID: TOB-SPL-30 The documentation specifies which functionalities should not be working when the system is paused, including the claiming of rewards; however, the claim function does not have the whenNotPaused modifier. As a result, users can claim their rewards even when the system is paused.

**Exploit Scenario**

Alice, who has the ROLE_PAUSER role in the system, pauses the protocol after she sees a possible vulnerability in the claim function. The Spool team believes there are no possible funds moving from the system; however, users can still claim their rewards.

**Recommendations**

Short term, add the whenNotPaused modifier to the claim function.

Long term, improve the system’s unit and integration tests by adding a test to verify that the expected functionalities do not work when the system is in a paused state.

## Users who deposit and then withdraw before doHardWork lose their tokens

**Description**

Users who deposit and then withdraw assets before doHardWork is called will receive zero tokens from their withdrawal operations.

When a user deposits assets, the depositAssets function mints an NFT with some metadata to the user who can later redeem it for the underlying SVT tokens.

Users call the claimSmartVaultTokens function in the SmartVaultManager contract to claim SVT tokens. It is important to note that this function calls the _syncSmartVault function with false as the last argument, which means that it will not revert if the current flush index and the flush index to sync are the same. Then, claimSmartVaultTokens delegates the work to the corresponding function in the DepositManager contract.

Later, the claimSmartVaultTokens function in DepositManager (figure 31.3) computes the SVT tokens that users will receive by calling the getClaimedVaultTokensPreview function and passing the bag.mintedSVTs value for the flush corresponding to the burned NFT.

Then, getClaimedVaultTokensPreview calculates the SVT tokens proportional to the amount deposited.

However, the value of _flushShares[smartVault][bag.data.flushIndex].mintedVaultShares , shown in f igure 31.3, will always be 0 : the value is updated in the syncDeposit function, but because the current flush cycle is not finished yet, syncDeposit cannot be called through syncSmartVault.

The same problem appears in the redeem , redeemFast , and claimWithdrawal functions.

**Exploit Scenario**

Bob deposits assets into a smart vault, but he notices that he deposited in the wrong smart vault. He calls redeem and claimWithdrawal , expecting to receive back his tokens, but he receives zero tokens. The tokens are locked in the smart contracts.

**Recommendations**

Short term, do not allow users to withdraw tokens when the corresponding flush has not yet happened.

Long term, document and test the expected effects when calling functions in all of the possible orders, and add adequate constraints to avoid unexpected behavior.

## Removal of a strategy could result in loss of funds


**Description**

A Spool admin can remove a strategy from the system, which will be replaced by a ghost strategy in all smart vaults that use it; however, if a strategy is removed when the system is in specific states, funds to be deposited or withdrawn in the next “do hard work” cycle will be lost.

If the following sequence of events occurs, the asset deposited will be lost from the removed strategy:

1. A user deposits assets into a smart vault.
2. The flush function is called. The StrategyRegistry._assetsDeposited[strategy][xxx][yyy] storage variable now has assets to send to the given strategy in the next “do hard work” cycle.
3. The strategy is removed.
4. doHardWork is called, but the assets for the removed strategy are locked in the master wallet because the function can be called only for valid strategies.

If the following sequence of events occurs, the assets withdrawn from a removed strategy will be lost:

1. doHardWork is called.
2. The strategy is removed before a smart vault sync is done.

**Exploit Scenario**

Multiple smart vaults use strategy A. Users deposited a total of $1 million, and $300,000 should go to strategy A. Strategy A is removed due to an issue in the third-party protocol. All of the $300,000 is locked in the master wallet.

**Recommendations**

Short term, modify the associated code to properly handle deposited and withdrawn funds when strategies are removed.

Long term, improve the system’s unit and integration tests: consider all of the possible transaction sequences in the system’s state and test them to ensure their correct behavior.

## Solidity compiler optimizations can be problematic

**Description**

Ajna protocol has enabled optional compiler optimizations in Solidity.

There have been several optimization bugs with security implications. Moreover, optimizations are actively being developed . Solidity compiler optimizations are disabled by default, and it is unclear how many contracts in the wild actually use them. Therefore, it is unclear how well they are being tested and exercised.

High-severity security issues due to optimization bugs have occurred in the past . A high-severity bug in the emscripten -generated solc-js compiler used by Truffle and Remix persisted until late 2018. The fix for this bug was not reported in the Solidity CHANGELOG. Another high-severity optimization bug resulting in incorrect bit shift results was patched in Solidity 0.5.6 . More recently, another bug due to the incorrect caching of keccak256 was reported.

A compiler audit of Solidity from November 2018 concluded that the optional optimizations may not be safe.

It is likely that there are latent bugs related to optimization and that new bugs will be introduced due to future optimizations.

**Exploit Scenario**

A latent or future bug in Solidity compiler optimizations—or in the Emscripten transpilation to solc-js —causes a security vulnerability in the Ajna protocol contracts.

**Recommendations**

Short term, measure the gas savings from optimizations and carefully weigh them against the possibility of an optimization-related bug.

Long term, monitor the development and adoption of Solidity compiler optimizations to assess their maturity.

## findIndexAndSumOfSums ignores global scalar

**Description**

The findIndexAndSumOfSums method ignores the global scalar of the scaled Fenwick tree while calculating the smallest index at which the prefix sum is at least the given target value.

In a scaled Fenwick tree, values at power-of-two indices contain the prefix sum of the entire underlying array up to that index. Similarly, scalars at power-of-two indices contain a scaling factor by which all lower-index values should be multiplied to get the correct underlying values and prefix sums.

The findIndexAndSumOfSums method performs a binary search starting from the middle power-of-two index, 4096 (2 12 ). If the prefix sum up to that point is too small, the algorithm checks higher indices, and if the prefix sum is too large, it checks lower indices. But the global scalar at index 8192 (2 13 ) is not visited by this method, and its value is never considered. If the global scalar contains a non-default value, then the indices and sums returned by findIndexAndSumOfSums will be incorrect.

**Exploit Scenario**

A subsequent update to the codebase allows global rescales in constant time by modifying the scale value at index 2 13 . As a result, findIndexAndSumOfSums returns incorrect values, causing auction and lending operations to malfunction.

Recommendations

Short term, initialize the runningScale variable in findIndexAndSumOfSums to the global scalar instead of to one wad.

Long term, for each system component, build out unit tests to cover all known edge cases and document them thoroughly. This will facilitate a review of the codebase and help surface other similar issues.

## findMechanismOfProposal could shadow an extraordinary proposal

**Description**

The findMechanismOfProposal function will shadow an existing extraordinary proposal if a standard proposal with the same proposal ID exists. That is, the function will report that a given proposal ID corresponds to a standard proposal, even though an extraordinary proposal with the same ID exists.

Proposal IDs for both types of proposals are generated by hashing the proposal arguments, which are the same for both proposals.

The findMechanismOfProposal function is also called from the state function, which reports the state of a given proposal by ID.

Depending on how the state view function is used, its use of the flawed findMechanismOfProposal function could cause problems in the front end or other smart contracts that integrate with the GrantFund contract.

**Exploit Scenario**

Alice creates an extraordinary proposal to request 10 million AJNA tokens to pay for something important. Mallory does not like the proposal and creates a standard proposal with the same arguments. The front end, which calls state() to view the state of any type of proposal, now returns the state of Mallory's standard proposal instead of Alice's extraordinary proposal.

**Recommendations**

Short term, redesign the findMechanismOfProposal function so that it does not shadow any proposal. For example, have the function return an array of two items that will indicate whether a standard and extraordinary proposal with that proposal ID exists.

Long term, consider all of the information that the front end and other integrating smart contracts might require to function correctly, and design the corresponding view functions in the smart contracts to fulfill those requirements.

## Solidity compiler optimizations can be problematic

**Description**

The Raft Finance contracts have enabled compiler optimizations. There have been several optimization bugs with security implications. Additionally, optimizations are actively being developed . Solidity compiler optimizations are disabled by default, and it is unclear how many contracts in the wild use them, so how well they are being tested and exercised is unknown.

High-severity security issues due to optimization bugs have occurred in the past. For example, a high-severity bug in the emscripten-generated solc-js compiler used by Truffle and Remix persisted until late 2018. The fix for this bug was not reported in the Solidity CHANGELOG . Another high-severity optimization bug resulting in incorrect bit shift results was patched in Solidity v0.5.6 . More recently, a bug due to the incorrect caching of Keccak-256 was reported.

A compiler audit of Solidity from November 2018 concluded that the optional optimizations may not be safe. It is likely that there are latent bugs related to optimization and that new bugs will be introduced due to future optimizations.

**Exploit Scenario**

A latent or future bug in Solidity compiler optimizations causes a security vulnerability in the Raft Finance contracts.

**Recommendations**

Short term, measure the gas savings from optimizations and carefully weigh them against the possibility of an optimization-related bug.

Long term, monitor the development and adoption of Solidity compiler optimizations to assess their maturity.


## Votes can be delegated to contracts

**Description**

Votes can be delegated to smart contracts. This behavior contrasts with the fact that FXS tokens can be locked only in whitelisted contracts. Allowing votes to be delegated to smart contracts could lead to unexpected behavior.

By default, smart contracts are unable to gain voting power; to gain voting power, they need to be explicitly whitelisted by the Frax Finance team in the veFXS contract.


This is the intended design of the voting escrow contract, as allowing smart contracts to vote would enable wrapped tokens and bribes.

The VeFxsVotingDelegation contract enables users to delegate their voting power to other addresses, but it does not contain a check for smart contracts. This means that smart contracts can now hold voting power, and the team is unable to disallow this.

**Exploit Scenario**

Eve sets up a contract that accepts delegated votes in exchange for rewards. The contract ends up owning a majority of the FXS voting power.

**Recommendations**

Short term, consider whether smart contracts should be allowed to hold voting power. If so, document this fact; if not, add a check to the VeFxsVotingDelegation contract to ensure that addresses receiving delegated voting power are not smart contracts.

Long term, when implementing new features, consider the implications of adding them to ensure that they do not lift constraints that were placed beforehand.

## Solidity compiler optimizations can be problematic

**Description**

Arcade has enabled optional compiler optimizations in Solidity. According to a November 2018 audit of the Solidity compiler, the optional optimizations may not be safe.

High-severity security issues due to optimization bugs have occurred in the past. A high-severity bug in the emscripten-generated solc-js compiler used by Truffle and Remix persisted until late 2018; the fix for this bug was not reported in the Solidity changelog. Another high-severity optimization bug resulting in incorrect bit shift results was patched in Solidity 0.5.6. Another bug due to the incorrect caching of Keccak-256 was reported.

It is likely that there are latent bugs related to optimization and that future optimizations will introduce new bugs.

**Exploit Scenario**

A latent or future bug in Solidity compiler optimizations—or in the Emscripten transpilation to solc-js—causes a security vulnerability in the Arcade contracts.

**Recommendations**

Short term, measure the gas savings from optimizations and carefully weigh them against the possibility of an optimization-related bug.

Long term, monitor the development and adoption of Solidity compiler optimizations to assess their maturity.

## Token bridge will receive and lock ether

**Description**

The token bridge’s entrypoint for deposits can receive ether, but the token bridge cannot retrieve it in any way.

Users deposit ERC20 tokens using the outboundTransfer* functions from the token bridge. An example is shown below:

This function will trigger the creation of a retryable ticket, so it needs funds to pay fees and gas. These fees can be paid using ether or some specific ERC20, but in different token bridge deployments that share the same interface. In the latter case, the entry function should not receive ether even though it is payable.

**Exploit Scenario**

A user accidentally provides ether to a token bridge associated with a rollup that uses a custom ERC20 token fee. The ether will be locked in the token bridge.

**Recommendations**

Short term, add a condition that checks the value provided into the outboundTransfer function, and have the function revert if the value is positive.

Long term, review how funds flow from the user to/from different components, and ensure that there are no situations where tokens can be trapped.

## Redeeming xToken and fToken simultaneously uses incorrect TWAP price

**Description**

The Market contract redeems either the leverage token (xToken) or fractional token (fToken) and is currently the only contract that can redeem tokens from the Treasury contract. However, the market role can be granted to contracts, which permits simultaneous redemption of both token types. If both tokens are redeemed at once, the xToken will receive the same price as if only an fToken were being redeemed. Following an update, a user may be able to manipulate the price to be maxPrice by depositing just a single fToken and consequently receive a greater amount of the base asset.

Currently, redeeming an xToken will use the minPrice from the TWAP (time-weighted average price) to calculate how much base asset the user will receive. However, the _fetchTwapPrice function does not handle how the price should be determined in cases where both assets are being redeemed at once. As such, the Treasury should prevent this.

**Exploit Scenario**

A new market role is granted to a contract that allows redeeming xToken and fToken simultaneously, allowing the xToken to be redeemed at the incorrect price.

**Recommendations**

Short term, require that only one token is redeemed at a time by adding additional validation or creating two distinct entry points for redeeming either token.

Long term, strictly validate inputs from external contracts (even trusted) to ensure upgrades or configuration do not allow an important check to be bypassed.

## Incorrect TWAP price may be used for calculations such as collateral ratio

**Description**

The swap state of the protocol uses the enum value, Action.None when it is loaded for operations aside from minting and redeeming. The implementation of _fetchTwapPrice does not revert when the action is Action.None and the TWAP price is invalid, allowing the invalid price to be used to compute the collateral ratio. Since other contracts rely on these values to perform important validations and calculations, it is important that the price is valid and accurate.

This also may affect the following calculations, isUnderCollateral and currentBaseTokenPrice , and other operations that use _loadSwapState(Action.None)

**Exploit Scenario**

The rebalancer pool contract permits liquidations because it uses a value of collateralRatio that is based on an invalid price. This causes users to be liquidated at an unfair price.

**Recommendations**

Short term, define and implement what price should be used if the price is invalid when the action is Action.None or revert, if appropriate.

Long term, ensure the integrity and correctness of price calculations, especially those related to minting/ burning and depositing/withdrawing.

## Updating strategy may cause users to lose funds during redemption

**Description**

Collateral in the Treasury contract may be deposited into a strategy contract that the admin can update at any time. Since the update does not prevent collateral from being left in the old strategy contract or reset the balance that tracks the amount deposited in the strategyUnderlying strategy, users may receive substantially less than they are owed upon redeeming xToken and fToken, even though the collateral is available in the old strategy contract. Note that users may specify a minimum amount out in the market contract, but an admin would have to intervene and handle recovering the strategy’s assets to make them available to the user.

The implementation of _transferBaseToken overrides the amount withdrawn when the strategy fails to return the remainder of the tokens owed to the user. Because strategyUnderlying may be the balance deposited to previous strategies, it is possible that the current strategy does not have sufficient balance to fulfill the deficit; however, the operation, strategyUnderlying minus _diff , will not overflow. This may cause the user to receive substantially less because the balances of the strategy contracts are not fully withdrawn before an admin updates the strategy contract.

**Exploit Scenario**

The admin updates the strategy while collateral is deposited. As a result, strategyUnderlying —the number of base tokens in the strategy—is greater than zero. If a user redeems via the Market contract and the Treasury cannot successfully withdraw sufficient funds from the new strategy, the users’ fToken and xToken will have been burned without receiving their share of the base asset.

**Recommendations**

Short term, prior to updating the strategy, ensure that all funds have been withdrawn into the Treasury and that strategyUnderlying is reset to 0.

Long term, investigate how to handle scenarios where the available collateral declines due to losses incurred by the strategy.

## Collateral ratio does not account underlying value of collateral in strategy

**Description**

When collateral is moved into the strategy, the totalBaseToken amount remains unchanged. This does not account for shortfalls and surpluses in the value of the collateral moved into the strategy, and as a result, the collateral ratio may be over-reported or under-reported, respectively.

The integration of strategies in the treasury has not been fully designed and has flaws as evidenced by TOB-ADFX-5 . This feature should be reconsidered and thoroughly specified prior to active use. For example, it is unclear whether the strategy’s impact on the collateral ratio is robust below 130%. It may benefit the stability of the system to prevent too much of the collateral from entering strategies during periods of high volatility or when the system is not overcollateralized by implementing a buffer, or a maximum threshold of the amount of collateral devoted to strategies.

Since totalBaseToken is not updated for strategies, it may not be reflected correctly in the amount of base token available to be harvested as reported by the harvestable function.

**Recommendations**

Short term, disable the use of strategies for active deployments and consider removing the functionality altogether until it is fully specified and implemented.

Long term, limit the number of features in core contracts and do not prematurely merge incomplete features.

## Rewards are withdrawn even if protocol is not sufficiently collateralized

**Description**

The treasury allows any excess base token amount beyond what is considered collateral to be distributed by calling the harvest function, harvest . Some of this token amount is sent to the rebalance pool as rewards. Since the rebalance pool’s purpose is to encourage the re-collateralization of treasury, it follows that temporarily withholding the rewards until the protocol is re-collateralized, or using them to cover shortfalls, may more effectively stabilize the system.

**Recommendations**

Short term, consider pausing reward harvesting when the collateral ratio is below the stability ratio and even using the rewards to compensate for shortfalls.

Long term, perform additional review of the incentive compatibility and economic sustainability of the system.

## Rebalance pool withdrawal silently fails

**Description**

The FxUSDShareableRebalancePool contract disallows withdrawals by commenting out the internal call, which will silently fail for external integrations and users. Instead, the call should revert and provide a clear error message explaining that withdrawing fToken is disabled.

**Recommendations**

Short term, revert with a custom error if the function, withdraw is called.

Long term, do not leave commented-out code and explicitly handle error cases.

## Treating fToken as $1 creates arbitrage opportunity and unclear incentives

**Description**

The Treasury contract treats fToken as $1 regardless of the price on reference markets. This means that anyone can buy fToken and immediately redeem for profit, and the protocol’s collateralization will decrease more than it would have if valued at market value.

During periods where the protocol’s collateral ratio is less than the stability ratio and fToken is valued at a premium, users may not readily redeem the fToken as expected because they will receive only $1 in collateral. This may delay or prevent re-collateralization, as minting fToken is one action that is anticipated to aid re-collateralization during stability mode.

The arbitrage opportunity may exacerbate issues such as TOB-ADFX-20 by making it profitable to purchase fToken on decentralized exchanges and redeem it in excess, but this requires further investigation.

**Recommendations**

Short term, implement monitoring for the price of fToken and unusual activity, and create an action in an incident response plan for these scenarios.

Long term, conduct further analysis and determine when it is favorable to the protocol to consider fToken worth $1 and when it may be risky. Design and implement mitigations, if necessary, and perform invariant testing on the changes.

## Rounding direction for deposits does not favor the protocol

**Description**

When users mint fToken and xToken, they must transfer the amount of base token obtained by maxMintableFToken and maxMinatableXToken , respectively, as collateral. Currently, these functions round the value of _maxBaseIn , the amount of collateral, down due to integer division. Thus, in some cases, the amount of collateral required may be less than the amount required if the same calculation were done using real numbers, and the protocol will receive less collateral than expected. This requires further investigation, but it would be best to specify rounding directions explicitly even if this issue is not currently exploitable.

**Recommendations**

Short term, determine whether the protocol should round up deposits and implement the appropriate behavior, avoiding rounding errors that may trap the system.

Long term, specify rounding directions explicitly and use fuzz testing to identify if it is possible to exacerbate the rounding issue and profit.

## Unclear economic sustainability of allowing user to avoid liquidations

**Description**

The FxUSD contracts allow moving fToken from the rebalance pool into them and minting FxUSD. (This method is used to attempt to increase the collateral ratio by burning fToken.) Moving tokens in this way is allowed even when liquidations are possible (i.e., the collateral ratio is sufficiently low). The rebalance pool is intended to socialize the losses among fToken depositors, but nothing prevents withdrawing prior to liquidation transactions. This does not ensure that all depositors are on the hook for liquidation.

Users who front-run liquidations and withdraw from the rebalance pool will not have the liquidated collateral credited to their share of rewards from the rebalance pool, but they will retain their balance of fToken. Since redeeming fToken may be the only action for re-collateralizating the protocol if xToken minting is failing ( TOB-ADFX-18 ), allowing withdrawals during stability mode may threaten the ability of the protocol to recover from delinquency.

**Recommendations**

Short term, consider preventing withdrawals from the rebalance pool when the fToken’s treasury’s collateral ratio is below the stability ratio and thus able to be liquidated.

Long term, perform additional economic analysis regarding liquidations and whether the incentives of fToken holders and FxUSD holders align with the protocol’s interests.

## Use of duplicate functions

**Description**

The ProteusLogic and Proteus contracts both contain a function used to update the internal slices array. Although calls to these functions currently lead to identical outcomes, there is a risk that a future update could be applied to one function but not the other, which would be problematic.

Using duplicate functions in different contracts is not best practice. It increases the risk of a divergence between the contracts and could significantly affect the system properties. Defining a function in one contract and having other contracts call that function is less risky.

**Exploit Scenario**

Alice, a developer of the Shell Protocol, is tasked with updating the ProteusLogic contract. The update requires a change to theProteus._updateSlicesfunction. However, Alice forgets to update theProteusLogic._updateSlicesfunction. Because of this omission, the functions’ updates to the internal slices array may produce different results.

**Recommendations**

Short term, select one of the two _updateSlices functions to retain in the codebase and to maintain going forward. 

Long term, consider consolidating the Proteus and ProteusLogic contracts into a single implementation, and avoid duplicating logic.



