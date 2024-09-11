## Test suite inconsistencies / failures

**Description**

The Balancer protocol tests experience inconsistent behavior when they are built and run.

The Balancer tests fail to run on certain machines with a fresh clone of the repository because they are unable to find certain artifacts (figure 2.1). An inability to consistently run tests across machines hinders the development and review processes.

Additionally, when it can be run, one of the Balancer tests intermittently fails (figure 2.2). Test failures such as these should always be investigated to ensure that they are not a sign of a bug in a codebase.

**Exploit Scenario**

Alice, a Balancer developer, refactors a portion of the code as part of an effort to simplify it. In doing so, she accidentally introduces unexpected system behavior that could cause a pool to enter an invalid state. Because the refactored code cannot be tested, the behavior is not detected before the system is deployed.

**Recommendations**

Short term, investigate all failing test cases to ensure that the code is behaving as expected.

Long term, implement comprehensive testing across the codebase to ensure that the code behaves as expected under all conditions.

## Broken test cases that hide security issues

**Description**

Multiple test cases do not check sufficient conditions to verify the correctness of the code, which could result in the deployment of buggy code in production and the loss of funds.

The test_extendRewardEmission_ok test does not check the new reward rate and duration to verify the effect of the call to the extendRewardEmission function on the RewardManager contract:

The test_removeReward_ok test does not check the new reward token count and the deletion of the reward configuration for the smart vault to verify the effect of the call to the removeReward function on the RewardManager contract:

There is no test case to check the access controls of the removeReward function. Similarly, the test_forceRemoveReward_ok test does not check the effects of the forced removal of a reward token. Findings TOB-SPL-28 and TOB-SPL-29 were not detected by tests because of these broken test cases. The test_removeStrategy_betweenFlushAndDHW test does not check the balance of the master wallet.

The test_removeStrategy_betweenFlushAndDhwWithdrawals test removes the strategy before the “do hard work” execution of the deposit cycle instead of removing it before the “do hard work” execution of the withdrawal cycle, making this test case redundant. Finding TOB-SPL-33 would have been detected if this test had been correctly implemented.

There may be other broken tests that we did not find, as we could not cover all of the test cases

**Exploit Scenario**

The Spool team deploys the protocol. After some time, the Spool team makes some changes in the code that introduces a bug that goes unnoticed due to the broken test cases. The team deploys the new changes with confidence in their tests and ends up introducing a security issue in the production deployment of the protocol.

**Recommendations**

Short term, fix the test cases described above.

Long term, review all of the system’s test cases and make sure that they verify the given state change correctly and sufficiently after an interaction with the protocol. Use Necessist to find broken test cases and fix them.

## Certain identity curve configurations can lead to a loss of pool tokens

**Description**

A rounding error in an integer division operation could lead to a loss of pool tokens and the dilution of liquidity provider (LP) tokens.

We reimplemented certain of Cowri Labs’s fuzz tests and used Echidna to test the system properties specified in the Automated Testing section. The original fuzz testing used a fixed amount of 100 tokens for the initial xBalance and yBalance values; after we removed that limitation, Echidna was able to break some of the invariants. The Shell Protocol team should identify the largest possible percentage decrease in pool utility or utility per shell (UPS) to better quantify the impact of a broken invariant on system behavior.

In some of the breaking cases, the ratio of token balances, m, was close to the X or Y asymptote of the identity curve. This means that an attacker might be able to disturb the balance of the pool (through flash minting or a large swap, for example) and then exploit the broken invariants.

**Exploit Scenario**

Alice withdraws USD 100 worth of token X from a Proteus-based liquidity pool by burning her LP tokens. She eventually decides to reenter the pool and to provide the same amount of liquidity. Even though the curve’s configuration is similar to the configuration at the time of her withdrawal, her deposit leads to only a USD 90 increase in the pool’s balance of token X; thus, Alice receives fewer LP tokens than she should in return, effectively losing money because of an arithmetic error.

**Recommendations**

Short term, investigate the root cause of the failing properties. Document and test the expected rounding direction (up or down) of each arithmetic operation, and ensure that the rounding direction used in each operation benefits the pool.

Long term, implement the fuzz testing recommendations outlined in appendix C.





