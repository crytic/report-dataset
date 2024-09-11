## Governance role is a single point of failure

**Description**

Because the governance role is centralized and responsible for critical functionalities, it constitutes a single point of failure within the Increment Protocol. The role can perform the following privileged operations:

● Whitelisting a perpetual market

● Setting economic parameters

● Updating price oracle addresses and setting fixed prices for assets

● Managing protocol insurance funds

● Updating the addresses of core contracts

● Adding support for new reserve tokens to the UA contract

● Pausing and unpausing protocol operations

  

These privileges give governance complete control over the protocol and therefore access to user and protocol funds. This increases the likelihood that the governance account will be targeted by an attacker and incentivizes governance to act maliciously. Note, though, that the governance role is currently controlled by a multisignature wallet (a multisig) and that control may be transferred to a decentralized autonomous organization (DAO) in the future.

  

**Exploit Scenario**

Eve, an attacker, creates a fake token, compromises the governance account, and adds the fake token as a reserve token for UA. She mints UA by making a deposit of the fake token and then burns the newly acquired UA tokens, which enables her to withdraw all USDC from the reserves.

  

**Recommendations**

Short term, minimize the privileges of the governance role and update the documentation to include the implications of those privileges. Additionally, implement reasonable time delays for privileged operations.

  

Long term, document an incident response plan and ensure that the private keys for the multisig are managed safely. Additionally, carefully evaluate the risks of moving from a multisig to a DAO and consider whether the move is necessary.

## Use of tx.origin for access controls

**Description**

The wallet factory uses tx.origin to authenticate users. This pattern is commonly considered insecure, and it could result in the creation of undesired wallets or wallets in a vulnerable state.

WalletFactory allows users to call the createAndCallTxOrigin function to create wallets whose owner is tx.origin

However, the user creating the transaction might not be aware that this function is executed, which would create an undesired wallet. Moreover, a malicious initial call to the   CallHelper.functionCall is executed in createAndCallTxOrigin, which could put the created wallet in a vulnerable state.

**Exploit Scenario**

Bob frequently creates nested wallets that hold USDC tokens. Eve notices this and tricks Bob into transferring a token with a callback, and she uses the callback to call createAndCallTxOrigin. During the wallet’s initialization, the wallet approves Eve to transfer all of its UDSC tokens. The new wallet is shown in the Nested Finance interface. Bob starts using the wallet. After some time, the wallet holds $10 million worth of USDC. Eve steals the tokens.

**Recommendations**

Short term, remove createAndCallTxOrigin.

Long term, never use tx.origin for authentication.

## DCA ownership transfers can be abused to steal tokens

**Description**

DCA ownership is transferred without the consent of the new owner, which means that DCA ownership transfers can be abused to steal users’ tokens.

The recipient of a DCA swap is stored in the swap parameter ISwapRouter.ExactInputSingleParams.recipient.

In most cases, the recipient of the swap will also be the creator of the DCA, stored in ownerOf[dcaId]. When an automated swap is performed, the tokens are transferred from the current owner of the given DCA to the recipient defined in the swap parameters.

When ownership of a DCA is transferred via the transferDcaOwnership function, the function sets the new owner without checking for the new owner’s consent. If ownership is transferred after the DCA has already been set up and started, then the tokens will be transferred from the new user instead.

This issue can be abused to steal tokens from any user that has already given their token-spending approval to the NestedDca contract.

**Exploit Scenario**

Bob sets up a DCA to convert his USDC tokens to ETH. For this purpose, he has given the NestedDca contract unlimited approval. He currently has $1 million worth of USDC in his wallet. Eve notices this and sets up a DCA that transfers $1 million of USDC (that she does not own) to ETH in a single swap. She then starts the DCA and immediately transfers the ownership of the DCA to Bob. Once Eve’s automated task is performed, Bob’s $1 million of USDC is transferred to Eve.

**Recommendations**

Short term, either disallow transfers of DCA ownership or add a two-step ownership transfer process, in which the new owner must accept the ownership transfer. Long term, create documentation/diagrams that highlight all of the transfers of assets and the token approvals. Add additional tests, and use Echidna to test system invariants targeting the assets and DCA transfers.

## Governance is a single point of failure

**Description**

Because the governance role is responsible for critical functionalities, it constitutes a single point of failure within the Loans and Revolving Credit Lines.

The role can perform the following privileged operations:

● Registering and deploying new products
● Setting economic parameters
● Entering the pool into a non-standard payment procedure
● Setting the phases of the Loans and Revolving Credit Lines
● Proposing and executing timelock operations
● Granting and revoking lender and borrower roles, respectively
● Changing the yield providers used by the Pool Custodian

These privileges give governance complete control over the protocol and critical protocol operations. This increases the likelihood that the governance account will be targeted by an attacker and incentivizes governance to act maliciously.

**Exploit Scenario**

Eve, a malicious actor, manages to take over the governance address. She then changes the yield providers to her own smart contract, effectively stealing all the funds in the Atlendis Labs protocol.

**Recommendations**

Short term, consider splitting the privileges across different addresses to reduce the impact of governance compromise. Ensure these powers and privileges are kept as minimal as possible.

Long term, document an incident response plan (see Appendix H) and ensure that the private keys for the multisig are managed safely (see Appendix G).

## Risk of SmartVaultFactory DoS due to lack of access controls on grantSmartVaultOwnership


**Description**

Anyone can set the owner of the next smart vault to be created, which will result in a DoS of the SmartVaultFactory contract.

The grantSmartVaultOwnership function in the SpoolAccessControl contract allows anyone to set the owner of a smart vault. This function reverts if an owner is already set for the provided smart vault.

The SmartVaultFactory contract implements two functions for deploying new smart vaults: the deploySmartVault function uses the create opcode, and the deploySmartVaultDeterministically function uses the create2 opcode. Both functions create a new smart vault and call the grantSmartVaultOwnership function to make the message sender the owner of the newly created smart vault.

Any user can pre-compute the address of the new smart vault for a deploySmartVault transaction by using the address and nonce of the SmartVaultFactory contract; to compute the address of the new smart vault for a deploySmartVaultDeterministically transaction, the user could front-run the transaction to capture the salt provided by the user who submitted it.

**Exploit Scenario**

Eve pre-computes the address of the new smart vault that will be created by the deploySmartVault function in the SmartVaultFactory contract. She then calls the grantSmartVaultOwnership function with the pre-computed address and a nonzero address as arguments. Now, every call to the deploySmartContract function reverts, making the SmartVaultFactory contract unusable.

Using a similar strategy, Eve blocks the deploySmartVaultDeterministically function by front-running the user transaction to set the owner of the smart vault address computed using the user-provided salt.

**Recommendations**

Short term, add the onlyRole(ROLE_SMART_VAULT_INTEGRATOR, msg.sender) modifier to the grantSmartVaultOwnership function to restrict access to it.

Long term, follow the principle of least privilege by restricting access to the functions that grant specific privileges to actors of the system.

## Issues with the management of access control roles in deployment script

**Description**

Finding ID: TOB-SPL-36 The deployment script does not properly manage or assign access control roles. As a result, the protocol will not work as expected, and the protocol’s contracts cannot be upgraded.

The deployment script has multiple issues regarding the assignment or transfer of access control roles. It fails to grant certain roles and to revoke temporary roles on deployment:

● Ownership of the ProxyAdmin contract is not transferred to an EOA, multisig wallet, or DAO after the system is deployed, making the smart contracts non-upgradeable.
● The DEFAULT_ADMIN_ROLE role is not transferred to an EOA, multisig wallet, or DAO after the system is deployed, leaving no way to manage roles after deployment.
● The ADMIN_ROLE_STRATEGY role is not assigned to the StrategyRegistry contract, which is required to grant the ROLE_STRATEGY role to a strategy contract. Because of this, new strategies cannot be registered.
● The ADMIN_ROLE_SMART_VAULT_ALLOW_REDEEM role is not assigned to the SmartVaultFactory contract, which is required to grant the ROLE_SMART_VAULT_ALLOW_REDEEM role to smartVault contracts.
● The ROLE_SMART_VAULT_MANAGER and ROLE_MASTER_WALLET_MANAGER roles are not assigned to the DepositManager and WithdrawalManager contracts, making them unable to move funds from the master wallet contract.

We also found that the ROLE_SMART_VAULT_ADMIN role is not assigned to the smart vault owner when a new smart vault is created. This means that smart vault owners will not be able to manage their smart vaults.

**Exploit Scenario**

The Spool team deploys the smart contracts using the deployment script, but due to the issues described in this finding, the team is not able to perform the role management and upgrades when required.

**Recommendations**

Short term, modify the deployment script so that it does the following on deployment:

 - ● Transfers ownership of the proxyAdmin contract to an EOA, multisig
   wallet, or DAO
   ● Transfers the DEFAULT_ADMIN_ROLE role to an EOA,
   multisig wallet, or DAO
   ● Grants the required roles to the smart
   contracts
   ● Allow the SmartVaultFactory contract to grant the
   ROLE_SMART_VAULT_ADMIN role to owners of newly created smart vaults

Long term, document all of the system’s roles and interactions between components that require privileged roles. Make sure that all of the components are granted their required roles following the principle of least privilege to keep the protocol secure and functioning as expected.

## Extraordinary proposal can be used to steal extraordinary amounts of AJNA

**Description**

The ExtraordinaryFunding contract's voteExtraordinary function accepts any address passed in for the account that will place the vote. As a result, an attacker can create an extraordinary proposal and call voteExtraordinary for each account that has voting power to make the proposal succeed (as long as the minimum threshold is adhered to).

To be able to place votes on both standard and extraordinary proposals, an account must call ajnaToken.delegate(address) to enable the protocol to track their AJNA token balance. If the passed-in address argument is that of the caller, then the caller will be allowed to vote (i.e., the caller delegates voting power to himself). On the other hand, if a different account's address is passed in, that account will receive the caller’s voting power (i.e., the caller delegates voting power to another account). To summarize, until an account calls ajnaToken.delegate(address) , that account's tokens cannot be used to place votes on any proposals.

An extraordinary proposal can be voted on by everyone that has voting power. Additionally, any account can place a vote only once, can vote only in favor of a proposal, can vote only with their entire voting power, and cannot undo a vote that has already been placed. An extraordinary proposal succeeds when there are enough votes in favor and the minimum threshold is adhered to. There is no minimum amount of time that needs to pass before an extraordinary proposal can be executed after it has gathered enough votes.

**Exploit Scenario**

Off-chain, Mallory collects a list of all of the accounts that have voting power. She sums all of the voting power and calculates off-chain the maximum number of AJNA tokens that could be withdrawn if a proposal gathered all of that voting power. Mallory writes a custom contract that, inside the constructor, creates an extraordinary proposal to transfer all of the AJNA tokens to an account she controls, loops through the collected accounts with voting power, and calls GrantFund.voteExtraordinary for each account, followed by a call to GrantFund.executeExtraordinary . Mallory deploys the contract and receives the AJNA tokens.

**Recommendations**

Short term, replace the account argument in the voteExtraordinary function with msg.sender.

Long term, develop a list of invariants for the grant fund system contracts and implement invariant testing to test that they hold.

## The relay function can be used to call critical safe functions

**Description**

The relay function of the FraxGovernorOmega contract supports arbitrary calls to arbitrary targets and can be leveraged in a proposal to call sensitive functions on the Gnosis Safe.

The FraxGovernorOmega contract checks proposed transactions to ensure they do not target critical functions on the Gnosis Safe contract outside of the more restrictive FraxGovernorAlpha flow.

A malicious user can hide a call to the Gnosis Safe by wrapping it in a call to the relay function. There are no further restrictions on the target contract argument, which means the relay function can be called with calldata that targets the Gnosis Safe contract.

**Exploit Scenario**

Alice, a veFXS holder, submits a transaction to the propose function. The targets array contains the FraxGovernorOmega address, and the corresponding calldatas array contains an encoded call to its relay function. The encoded call to the relay function has a target address of an allowlisted Gnosis Safe and an encoded call to its approveHash function with a payload of a malicious transaction hash. Due to the low quorum threshold on FraxGovernorOmega and the shorter voting period, Alice is able to push her malicious transaction through, and it is approved by the safe even though it should not have been.

**Recommendations**

Short term, add a check to the relay function that prevents it from targeting addresses of allowlisted safes.

Long term, carefully examine all cases of user-provided inputs, especially where arbitrary targets and calldata can be submitted, and expand the unit tests to account for edge cases specific to the wider system.

## Risk of funds becoming trapped if owner key is lost before raffle settlement

**Description**

Due to overly restrictive access controls on the functions used to settle the auction and raffle, in the event that access to the owner key is lost before these are settled, there will be no way for users to reclaim their funds, even for unsuccessful bids.

A previous version of the AuctionRaffle contract required a random seed value when calling settleRaffle , so the settlement functions included the onlyOwner modifier. The current version of the contract relies on Chainlink’s Verifiable Random Function (VRF) service to request randomness on-chain as part of the raffle settlement flow, and neither the settleAuction or settleRaffle functions take any parameters (figure 1.1).

In order for users to recover their funds (for the “golden ticket” winner of the raffle, funds in excess of the reserve price for other raffle winners, and all other unsuccessful bidders), the contract must be in the RAFFLE_SETTLED state, which is the state the contract remains in until the claiming period closes (figure 1.2).

In the unlikely event that the team managing the AuctionRaffle contract loses access to the contract’s owner key before settling the raffle, according to figure 1.2, the contract state machine will be unable to progress until the current time reaches the value in the _claimingEndTime variable. After that point, the only notable function in the contract is withdrawUnclaimedFunds (figure 1.3), which can be called only by the contract’s owner. As a result, any funds escrowed in the contract as part of the auction and raffle will be unrecoverable.

**Exploit Scenario**

The team loses access to the owner key of the AuctionRaffle contract late in the bidding process. The auction and raffle can no longer be settled due to the onlyOwner modifier on the functions that trigger these state transitions, so the contract will not enter the claim period. As a result, all of the funds for successful and unsuccessful bids will be considered “unclaimed” and will be recoverable by the contract owner only if access to the owner key is regained.

**Recommendations**

Short term, remove the onlyOwner modifier from the settleAuction and settleRaffle functions in the AuctionRaffle contract. This will allow the system to progress through all of its states without owner intervention once deployed.

## LiquidityPoolProxy owners can steal user funds

Description TheLiquidityPoolProxycontract implements theIOceanPrimitiveinterface and can integrate with theOceancontract as a primitive.The proxy contract calls into an implementation contract to perform deposit, swap, and withdrawal operations (figure 2.1).

However, the owner of aLiquidityPoolProxycontractcan perform the privileged operation of changing the underlying implementation contract via a call to setImplementation(figure 2.2). The owner could thusreplace the underlying implementation with a malicious contract to steal user funds.

This level of privilege creates a single point of failure in the system. It increases the likelihood that a contract’s owner will be targeted by an attacker and incentivizes the owner to act maliciously.

**Exploit Scenario**

Alice deploys a LiquidityPoolProxy contract as an Ocean primitive. Eve gains access to Alice’s machine and upgrades the implementation to a malicious contract that she controls. Bob attempts to swap USD 1 million worth of shDAI for shUSDC by calling computeOutputAmount. Eve’s contract returns 0 for outputAmount. As a result, the malicious primitive’s balance of shDAI increases by USD 1 million, but Bob does not receive any tokens in exchange for his shDAI.

**Recommendations**

Short term, document the functions and implementations that LiquidityPoolProxy contract owners can change. Additionally, split the privileges provided to the owner role across multiple roles to ensure that no one address has excessive control over the system.

Long term, develop user documentation on all risks associated with the system, including those associated with privileged users and the existence of a single point of failure.


