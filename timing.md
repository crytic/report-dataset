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
