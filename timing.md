## Ownership transfers can be front-run

  

**Description**

The PerpOwnable contract provides an access control mechanism for the minting and burning of a Perpetual contract’s vBase or vQuote tokens. The owner of these token contracts is set via the transferPerpOwner function, which assigns the owner’s address to the perp state variable. This function is designed to be called only once, during deployment, to set the Perpetual contract as the owner of the tokens. Then, as the tokens’ owner, the Perpetual contract can mint / burn tokens during liquidity provisions, trades, and liquidations. However, because the function is external, anyone can call it to set his or her own malicious address as perp, taking ownership of a contract’s vBase or vQuote tokens.

  

If the call were front-run, the Perpetual contract would not own the vBase or vQuote tokens, and any attempts to mint / burn tokens would revert. Since all user interactions require the minting or burning of tokens, no liquidity provisions, trades, or liquidations would be possible; the market would be effectively unusable. An attacker could launch such an attack upon every perpetual market deployment to cause a denial of service (DoS).

  

**Exploit Scenario**

Alice, an admin of the Increment Protocol, deploys a new Perpetual contract. Alice then attempts to call transferPerpOwner to set perp to the address of the deployed contract. However, Eve, an attacker monitoring the mempool, sees Alice’s call to transferPerpOwner and calls the function with a higher gas price. As a result, Eve gains ownership of the virtual tokens and renders the perpetual market useless. Eve then repeats the process with each subsequent deployment of a perpetual market, executing a DoS attack.

  

**Recommendations**

Short term, move all functionality from the PerpOwnable contract to the Perpetual contract. Then add the hasRole modifier to the transferPerpOwner function so that the function can be called only by the manager or governance role.

  

Long term, document all cases in which front-running may be possible, along with the implications of front-running for the codebase.

  
  
  

## Liquidations are vulnerable to sandwich attacks

  

**Description**

Token swaps that are performed to liquidate a position use a hard-coded zero as the “minimum-amount-out” value, making them vulnerable to sandwich attacks.

  

The minimum-amount-out value indicates the minimum amount of tokens that a user will receive from a swap. The value is meant to provide protection against pool illiquidity and sandwich attacks. Senders of position and liquidity provision updates are allowed to specify a minimum amount out. However, the minimum-amount-out value used in liquidations of both traders’ and liquidity providers’ positions is hard-coded to zero. Figures 9.1 and 9.2 show the functions that perform these liquidations (_liquidateTrader and _liquidateLp, respectively).

  

Without the ability to set a minimum amount out, liquidators are not guaranteed to receive any tokens from the pool during a swap. If a liquidator does not receive the correct amount of tokens, he or she will be unable to close the position, and the transaction will revert; the revert will also prolong the Increment Protocol’s exposure to debt. Moreover, liquidators will be discouraged from participating in liquidations if they know that they may be subject to sandwich attacks and may lose money in the process.

  

**Exploit Scenario**

Alice, a liquidator, notices that a position is no longer valid and decides to liquidate it. When she sends the transaction, the protocol sets the minimum-amount-out value to zero. Eve’s sandwich bot identifies Alice’s liquidation as a pure profit opportunity and sandwiches it with transactions. Alice’s liquidation fails, and the protocol remains in a state of debt.

  

**Recommendations**

Short term, allow liquidators to specify a minimum-amount-out value when liquidating the positions of traders and liquidity providers.

  

Long term, document all cases in which front-running may be possible, along with the implications of front-running for the codebase.

## GVault withdrawals from ConvexStrategy are vulnerable to sandwich attacks

**Description**

Token swaps that may be executed during vault withdrawals are vulnerable to sandwich attacks. Note that this is applicable only if a user withdraws directly from the GVault, not through the GRouter contract.

The ConvexStrategy contract performs token swaps through Uniswap V2, Uniswap V3, and Curve. All platforms allow the caller to specify the minimum-amount-out value, which indicates the minimum amount of tokens that a user wishes to receive from a swap. This provides protection against illiquid pools and sandwich attacks. Many of the swaps that the ConvexStrategy contract performs have the minimum-amount-out value hardcoded to zero. But a majority of these swaps can be triggered only by a Gelato keeper, which uses a private channel to relay all transactions. Thus, these swaps cannot be sandwiched.

However, this is not the case with the ConvexStrategy.withdraw function. The withdraw function will be called by the GVault contract if the GVault does not have enough tokens for a user withdrawal. If the balance is not sufficient, ConvexStrategy.withdraw will be called to retrieve additional assets to complete the withdrawal request. Note that the transaction to withdraw assets from the protocol will be visible in the public mempool (figure 6.1).

In the situation where the _amount that needs to be withdrawn is more than or equal to the total number of assets held by the contract, the withdraw function will call sellAllRewards and divestAll with _slippage set to false (see the highlighted portion of figure 6.1). The sellAllRewards function, which will call _sellRewards, sells all the additional reward tokens provided by Convex, its balance of CRV, and its balance of CVX for WETH. All these swaps have a hardcoded value of zero for the minimum-amount-out. Similarly, if _slippage is set to false when calling divestAll, the swap specifies a minimum-amount-out of zero.

By specifying zero for all these token swaps, there is no guarantee that the protocol will receive any tokens back from the trade. For example, if one or more of these swaps get sandwiched during a call to withdraw, there is an increased risk of reporting a loss that will directly affect the amount the user is able to withdraw.

**Exploit Scenario**

Alice makes a call to withdraw to remove some of her funds from the protocol. Eve notices this call in the public transaction mempool. Knowing that the contract will have to sell some of its rewards, Eve identifies a pure profit opportunity and sandwiches one or more of the swaps performed during the transaction. The strategy now has to report a loss, which results in Alice receiving less than she would have otherwise.

**Recommendations**

Short term, for _sellRewards, use the same minAmount calculation as in divestAll but replace debt with the contract’s balance of a given reward token. This can be applied for all swaps performed in _sellRewards. For divestAll, set _slippage to true instead of false when it is called in withdraw.

Long term, document all cases in which front-running may be possible and its implications for the codebase. Additionally, ensure that all users are aware of the risks of front-running and arbitrage when interacting with the GSquared system.

## Protocol migration is vulnerable to front-running and a loss of funds

**Description**

The migration from Gro protocol to GSquared protocol can be front-run by manipulating the share price enough that the protocol loses a large amount of funds.

The GMigration contract is responsible for initiating the migration from Gro to GSquared. The GMigration.prepareMigration function will deposit liquidity into the three-pool and then attempt to deposit the 3CRV LP token into the GVault contract in exchange for G3CRV shares (figure 12.1). Note that this migration occurs on a newly deployed GVault contract that holds no assets and has no supply of shares.

However, this prepareMigration function call is vulnerable to a share price inflation attack. As noted in this issue, the end result of the attack is that the shares (G3CRV) that the GMigration contract will receive can redeem only a portion of the assets that were originally deposited by GMigration into the GVault contract. This occurs because the first depositor in the GVault is capable of manipulating the share price significantly, which is compounded by the fact that the deposit function in GVault rounds in favor of the protocol due to a division in convertToShares (see TOB-GRO-11).

**Exploit Scenario**

Alice, a GSquared developer, calls prepareMigration to begin the process of migrating funds from Gro to GSquared. Eve notices this transaction in the public mempool, and front-runs it with a small deposit and a large token (3CRV) airdrop. This leads to a significant change in the share price. The prepareMigration call completes, but GMigration is left with a small, insufficient amount of shares because it has suffered from truncation in the convertToShares function. These shares can be redeemed for only a portion of the original deposit.

**Recommendations**

Short term, perform the GSquared system deployment and protocol migration using a private relay. This will mitigate the risk of front-running the migration or price share manipulation.

Long term, implement the short- and long-term recommendations outlined in TOB-GRO-11. Additionally, implement an ERC4626Router similar to Fei protocol’s implementation so that a minimum-amount-out can be specified for deposit, mint, redeem, and withdraw operations.

## Arbitrage opportunity in the PSM contract

**Description**

Given two PSM contracts for two different stablecoins, users could take advantage of the difference in price between the two stablecoins to engage in arbitrage.

This arbitrage opportunity exists because each PSM contract, regardless of the underlying stablecoin, holds that 1 MONO is worth $1. Therefore, if 100 stablecoin tokens are deposited into a PSM contract, the contract would mint 100 MONO tokens regardless of the price of the collateral token backing MONO.

The PolyMinter contract is vulnerable to the same arbitrage opportunity.

**Exploit Scenario**

USDC is trading at $0.98, and USDT is trading at $1. A user deposits 10,000 USDC tokens into the PSM contract and receives 10,000 MONO, which is worth $10,000, assuming there are no minting fees. The user then burns the 10,000 MONO he received from depositing the USDC, and he receives $9,990 worth of USDT in exchange, assuming there is a 0.1% redemption fee. Therefore, the arbitrageur was able to make a risk-free profit of $190.

**Recommendations**

Short term, document all front-running and arbitrage opportunities throughout the protocol, and ensure that users are aware of them. As development continues, reassess the risks associated with these opportunities and the adverse effects they could have on the protocol.

Long term, use an off-chain monitoring solution to detect any anomalous price fluctuations for the supported collateral tokens. Additionally, develop an incident response plan to ensure that any issues that arise can be addressed promptly and without confusion (see appendix E).

## Initialization functions can be front-run

**Description**

The FactoryRegistry contract is designed to register product factories and deploy products. As an access control mechanism, the initialize function transfers ownership to the governance parameter, allowing governance to execute owner-privileged actions.

However, the initialization function can be front-run by an attacker. An attacker can monitor the public mempool for the transaction that will call the initialize function and send their own transaction with a higher gas cost, calling the initialize function with their controlled address.

**Exploit Scenario**

Alice, the deployer of the FactoryRegistry contract, calls initialize, passing in the address of the governance contract. Eve notices her call in the mempool and front-runs her, calling initialize with her own address. As a result, the legitimate governance contract cannot register a new factory or deploy any products. Alice will need to redeploy.

**Recommendations**

Short term, ensure that the deployment scripts have robust protections against front-running attacks.

Long term, create an architecture diagram that includes the system’s functions and their arguments to identify any functions that can be front-run. Additionally, document all cases in which front-running may be possible, along with the effects of front-running on the codebase.

## Pool is put in NON_STANDARD state only after executeTimelock() is called

**Description**

In the event of a protocol error, the RCLGovernance contract has the ability to start a non-standard payment procedure or a rescue operation. This will instead use the NonStandardPaymentModule to handle the repayment procedure, or allow the protocol to transfer all the funds to the timelock recipient.

Since this is a sensitive operation, the governance contract must go through a delay using a time-locked procedure. The first step of the procedure is to call the startNonStandardPaymentProcedure/startRescueProcedure function, which will call the initiate function on the Timelock contract. After a delay, the executeTimelock function will be called, and all the funds will be transferred to the timelock.recipient. The pool will then be set to the NON_STANDARD state, disabling any borrowing/lending functionality.

However, the pool will be set to the NON_STANDARD state only after the executeTimelock function has been called. This means that, during the delay between the call to initiate and the call to executeTimelock, the functionality of the pool contract will be the same. This could allow an attacker to exploit the contracts.

**Exploit Scenario**

An issue is discovered within the RCL products. As a result, governance proposes the rescue operation and calls startRescueProcedure. However, Eve notices the call and finds a bug allowing her to drain the lenders’ funds. Because the pool is not in the NON_STANDARD state, Eve can steal all the assets from the protocol.

**Recommendations**

Short term, create a new phase that allows lenders to withdraw assets during an emergency, and put the pool in this phase when starting a time-locked procedure.

Long term, document an incident response plan that indicates the actions taken in the event of protocol failure.

## Risks with transaction reordering

**Description**

Throughout the codebase, there are opportunities to reorder transactions to maximize the borrowable amounts of the borrower, grief lender actions, or front-run borrow actions.

● A user could monitor the mempool for a borrow action and deposit their own assets into the pool at a lower rate than other lenders. By doing so, the user could ensure that their position would earn the maximum amount of interest. ● A miner/validator could reorder transactions in a block such that their deposit is complete before the borrow action has executed, while the other lender’s deposits could be moved to follow the borrow action. This would ensure that the miner/validator’s position earns the maximum amount of interest.
● If a lender wants to exit or detach their position, a borrower could notice the transaction in the mempool and front-run them with another borrow transaction, preventing them from withdrawing some or all of their assets.

Since the time of the deposit has no influence on the order in which the deposits will be borrowed (as long as they are in the same interest rate and epoch), lenders are incentivized to deposit their assets as close to the borrow action as possible.

**Exploit Scenario**

A Revolving Credit Line is created for Alice with a maximum borrowable amount of 100 ether. Bob deposits 20 ether into the pool, and Alice borrows 10 ether. After some time has passed, Bob decides to detach his position to withdraw the unborrowed amount. Alice sees this transaction in the mempool and front-runs it so Bob’s position ends up being fully borrowed, preventing him from withdrawing the assets.

**Recommendations**

Short term, analyze opportunities for transaction reordering within the current codebase and examine the potential implications of reordering state-changing actions. 

Long term, thoroughly document the risks of transaction reordering, or refactor the codebase to mitigate them if possible.

## Race condition in FraxGovernorOmega target validation

**Description**

The FraxGovernorOmega contract is intended for carrying out day-to-day operations and less sensitive proposals that do not adjust system governance parameters. Proposals directly affecting system governance are managed in the FraxGovernorAlpha contract, which has a much higher quorum requirement (40%, compared with FraxGovernorOmega ’s 4% quorum requirement). When a new proposal is submitted to the FraxGovernorOmega contract through the propose or addTransaction function, the target address of the proposal is checked to prevent proposals from interacting with sensitive functions in allowlisted safes outside of the higher quorum flow (figure 1.1). However, if a proposal to allowlist a new safe is pending in FraxGovernorAlpha , and another proposal that interacts with the pending safe is preemptively submitted through FraxGovernorOmega.propose , the proposal would pass this check, as the new safe would not yet have been added to the allowlist.

This issue provides a short window of time in which a proposal to update governance parameters that is submitted through FraxGovernorOmega could pass with the contract’s 4% quorum, rather than needing to go through FraxGovernorAlpha and its 40% quorum, as intended. Such an exploit would also require cooperation from the safe owners to execute the approved transaction. As the vast majority of operations in the FraxGovernorOmega process will be optimistic proposals, the community may not monitor the contract as comprehensively as FraxGovernorAlpha , and a minority group of coordinated veFXS holders could take advantage of this loophole.

**Exploit Scenario**

A FraxGovernorAlpha proposal to add a new Gnosis Safe to the allowlist is being voted on. In anticipation of the proposal’s approval, the new safe owner prepares and signs a transaction on this new safe for a contentious or previously vetoed action. Alice, a veFXS holder, uses FraxGovernorOmega.propose to initiate a proposal to approve the hash of this transaction in the new safe. Alice coordinates with enough other interested veFXS holders to reach the required quorum on the proposal. The proposal passes, and the new safe owner is able to update governance parameters without the consensus of the community.

**Recommendations**

Short term, add additional validation to the end of the proposal lifecycle to detect whether the target has become an allowlisted safe.

Long term, when designing new functionality, consider how this type of time-of-check to time-of-use mismatch could affect the system.

## Initialization functions vulnerable to front-running

**Description**

Several implementation contracts have initialization functions that can be front-run, which would allow an attacker to incorrectly initialize the contracts.

Due to the use of the delegatecall proxy pattern, the RootERC20Predicate and RootERC20PredicateFlowRate contracts (as well as other upgradeable contracts that are not in scope) cannot be initialized with a constructor, so they have initialize functions:

An attacker could front-run these functions and initialize the contracts with malicious values.

The documentation provided by the Immutable team indicates that they are aware of this issue and how to mitigate it upon deployment of the proxy or when upgrading the implementation. However, there do not appear to be any deployment scripts to demonstrate that this will be correctly done in practice, and the codebase’s tests do not cover upgradeability.

**Exploit Scenario**

Bob deploys the RootERC20Predicate contract. Eve deploys an upgradeable version of the ExitHelper contract and front-runs the RootERC20Predicate initialization, passing her contract’s address as the exitHelper argument. Due to a lack of post-deployment checks, this issue goes unnoticed and the protocol functions as intended for some time, drawing in a large amount of deposits. Eve then upgrades the ExitHelper contract to allow her to arbitrarily call the onL2StateReceive function of the RootERC20Predicate contract, draining all assets from the bridge.

**Recommendations**

Short term, either use a factory pattern that will prevent front-running the initialization, or ensure that the deployment scripts have robust protections against front-running attacks.

Long term, carefully review the Solidity documentation , especially the “Warnings” section, and the pitfalls of using the delegatecall proxy pattern.

## Risk of sandwich attacks

**Description**

The Proteus liquidity pool implementation does not use a parameter to prevent slippage. Without such a parameter, there is no guarantee that users will receive any tokens in a swap.

The LiquidityPool contract’s computeOutputAmount function returns an outputAmount value indicating the number of tokens a user should receive in exchange for the inputAmount. Many AMM protocols enable users to specify the minimum number of tokens that they would like to receive in a swap. This minimum number of tokens (indicated by a slippage parameter) protects users from receiving fewer tokens than expected.

As shown in figure 3.1, the computeOutputAmount function signature includes a 32-byte metadata field that would allow a user to encode a slippage parameter.

However, this field is not used in swaps (figure 3.2) and thus does not provide any protection against excessive slippage. By using a bot to sandwich a user’s trade, an attacker could increase the slippage incurred by the user and profit off of the spread at the user’s expense.

**Exploit Scenario**

Alice wishes to swap her shUSDC for shwETH. Because the computeOutputAmount function’s metadata field is not used in swaps to prevent excessive slippage, the trade can be executed at any price. As a result, when Eve sandwiches the trade with a buy and sell order, Alice sells the tokens without purchasing any, effectively giving away tokens for free.

**Recommendations** 

Short term, document the fact that protocols that choose to use the Proteus AMM engine should encode a slippage parameter in the metadata field. The use of this parameter will reduce the likelihood of sandwich attacks against protocol users. 

Long term, ensure that all calls to computeOutputAmount and computeInputAmount use slippage parameters when necessary, and consider relying on an oracle to ensure that the amount of slippage that users can incur in trades is appropriately limited.

