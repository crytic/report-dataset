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
