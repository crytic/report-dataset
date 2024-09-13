## Risks associated with oracle outages

**Description**

Under extreme market conditions, the Chainlink oracle may cease to work as expected, causing unexpected behavior in the Increment Protocol.

Such oracle issues have occurred in the past. For example, during the LUNA market crash, the Venus protocol was exploited because Chainlink stopped providing up-to-date prices. The interruption occurred because the price of LUNA dropped below the minimum price (minAnswer) allowed by the LUNA / USD price feed on the BNB chain. As a result, all oracle updates reverted. Chainlink’s automatic circuit breakers, which pause price feeds during extreme market conditions, could pose similar problems.


Note that these kinds of events cannot be tracked on-chain. If a price feed is paused, updatedAt will still be greater than zero, and answeredInRound will still be equal to roundID.

Thus, the Increment Finance team should implement an off-chain monitoring solution to detect any anomalous behavior exhibited by Chainlink oracles. The monitoring solution should check for the following conditions and issue alerts if they occur, as they may be indicative of abnormal market events:

  - An asset price that is approaching the minAnswer or maxAnswer value
   
  - The suspension of a price feed by an automatic circuit breaker
   
  - Any large deviations in the price of an asset

## Risk of oracle outages

**Description**

Under extreme market conditions, the Chainlink oracle may cease to work as expected, causing unexpected behavior in the Fraxlend protocol.

Such oracle issues have occurred in the past. For example, during the LUNA market crash, the Venus protocol was exploited because Chainlink stopped providing up-to-date prices. The interruption occurred because the price of LUNA dropped below the minimum price (minAnswer) allowed by the LUNA/USD price feed on the BNB chain. As a result, all oracle updates reverted. Chainlink’s automatic circuit breakers, which will pause price feeds during extreme market conditions, could pose similar problems.

Note that these kinds of events cannot be tracked on-chain. If a price feed is paused, updatedAt will still be greater than zero, and answeredInRound will still be equal to roundId.

Therefore, the Frax Finance team should implement an off-chain monitoring solution to detect any anomalous behavior exhibited by Chainlink oracles.

**Recommendations**

Short term, implement an off-chain monitoring solution that checks for the following conditions and issues alerts if they occur, as they may be indicative of abnormal market events:
- An asset price that is approaching the minAnswer or maxAnswer value
- The suspension of a price feed by an automatic circuit breaker
- Any large deviations in the price of an asset

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
