## Governance role is a single point of failure

**Description**

Because the governance role is centralized and responsible for critical functionalities, it constitutes a single point of failure within the Increment Protocol. The role can perform the following privileged operations:

- Whitelisting a perpetual market

- Setting economic parameters

- Updating price oracle addresses and setting fixed prices for assets

- Managing protocol insurance funds

- Updating the addresses of core contracts

- Adding support for new reserve tokens to the UA contract

- Pausing and unpausing protocol operations

  

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

- Registering and deploying new products
- Setting economic parameters
- Entering the pool into a non-standard payment procedure
- Setting the phases of the Loans and Revolving Credit Lines
- Proposing and executing timelock operations
- Granting and revoking lender and borrower roles, respectively
- Changing the yield providers used by the Pool Custodian

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
