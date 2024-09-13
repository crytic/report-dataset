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
- maxDeposit: MUST factor in both global and user-specific limits. For example, if deposits are entirely disabled (even temporarily), it MUST return 0.
- maxMint: MUST factor in both global and user-specific limits. For example, if mints are entirely disabled (even temporarily), it MUST return 0.

The current implementation of maxDeposit and maxMint in the Pool contract directly call and return the result of the same functions in PoolManager (figure 6.1). As shown in figure 6.1, both functions rely on _getMaxAssets, which correctly checks that the liquidity cap has not been reached and that deposits are allowed and otherwise returns 0. However, these checks are insufficient.

The deposit and mint functions have a checkCall modifier that will call the canCall function in the PoolManager to allow or disallow the action. This modifier first checks if the global protocol pause is active; if it is not, it will perform additional checks in _canDeposit. For this issue, it will be impossible to deposit or mint if the Pool is not active.

The maxDeposit and maxMint functions should return 0 if the global protocol pause is active or if the Pool is not active; however, these cases are not considered.

**Exploit Scenario**

A third-party protocol wants to deposit into Maple’s pool. It first calls maxDeposit to obtain the maximum amount of asserts it can deposit and then calls deposit. However, the latter function call will revert because the protocol is paused.

**Recommendations**

Short term, return 0 in maxDeposit and maxMint if the protocol is paused or if the pool is not active.

Long term, maintain compliance with the EIP specification being implemented (in this case, EIP-4626).
