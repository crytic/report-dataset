## Project dependencies contain vulnerabilities

  

**Description**

Although dependency scans did not identify a direct threat to the project under review, yarn audit identified dependencies with known vulnerabilities. Due to the sensitivity of the deployment code and its environment, it is important to ensure that dependencies are not malicious. Problems with dependencies in the JavaScript community could have a significant effect on the repository under review.

  

**Exploit Scenario**

Alice installs the dependencies of the in-scope repository on a clean machine. Unbeknownst to Alice, a dependency of the project has become malicious or exploitable. Alice subsequently uses the dependency, disclosing sensitive information to an unknown actor.

  

**Recommendations**

Short term, ensure that the Increment Protocol dependencies are up to date. Several node modules have been documented as malicious because they execute malicious code when installing dependencies to projects. Keep modules current and verify their integrity after installation.

  

Long term, integrate automated dependency auditing into the development workflow. If a dependency cannot be updated when a vulnerability is disclosed, ensure that the code does not use and is not affected by the vulnerable functionality of the dependency.

## Use of outdated libraries

**Description**

Trail of Bits used npm-check-updates to detect outdated dependencies in the codebase. This check found that 25 of the packages referenced by the package.json file are outdated. The following table lists only the dependencies that have not been updated to the latest major version.

**Recommendations**

Short term, update build process dependencies to the latest versions wherever possible. Use tools such as retire.js, node audit, and yarn audit to confirm that no vulnerable dependencies remain in the codebase.

Long term, implement dependency checks as part of the CI / CD pipeline. Do not allow builds to continue with outdated dependencies.

## Use of older versions of external libraries

**Description**

Finding ID: TOB-AJNA-5 The Ajna protocol depends on several external libraries, most notably OpenZeppelin and PRBMath, for various token interfaces and fixed-point math operations. However, the protocol uses outdated versions of these libraries.

The Ajna protocol uses version 2.4.3 of PRBMath, while the most recent version is 3.3.2, and version 4.7.0 of OpenZeppelin, which has one bug regarding compact signature malleability. Older versions of libraries may contain latent bugs that have been patched in newer versions. Using libraries that are not up to date is error-prone and could result in downstream issues.

The newer releases of both PRBMath and OpenZeppelin fix security issues that do not affect the current Ajna protocol smart contracts.

**Exploit Scenario**

A latent bug in version 2.4.3 of PRBMath causes fixed-point operations to be computed incorrectly. As a result, the precision loss throughout the protocol causes users to lose funds.

**Recommendations**

Short term, replace the use of the outdated versions of these libraries with their most recent versions.

Long term, set up automated monitoring of external library releases. Review each new release to see if it fixes a security issue that affects the Ajna contracts. Given that the Ajna contracts are not upgradeable, develop a plan for mitigating security issues that were found in library versions used by the protocol and then fixed in newer versions after the Ajna contracts have already been deployed.


## Vulnerable project dependency


**Description**

Finding ID: TOB-FRAXGOV-2 Although dependency scans did not uncover a direct threat to the project codebase, npm audit identified a dependency with a known vulnerability, the yaml library. Due to the sensitivity of the deployment code and its environment, it is important to ensure that dependencies are not malicious. Problems with dependencies in the JavaScript community could have a significant effect on the project system as a whole. 

**Exploit Scenario**

Alice installs the dependencies for the Frax Finance governance protocol, including the vulnerable yaml library, on a clean machine. She subsequently uses the library, which fails to throw an error while formatting a yaml configuration file, impacting important data that the system needs to run as intended.

**Recommendations**

Short term, ensure that dependencies are up to date. Several node modules have been documented as malicious because they execute malicious code when installing dependencies to projects. Keep modules current and verify their integrity after installation.

Long term, consider integrating automated dependency auditing into the development workflow. If dependencies cannot be updated when a vulnerability is disclosed, ensure that the project codebase does not use and is not affected by the vulnerable functionality of the dependency.

## Lack of public documentation regarding voting power expiry

**Description**

The user documentation concerning the calculation of voting power is unclear.

The Frax-Governance specification sheet provided by the Frax Finance team states, “Voting power goes to 0 at veFXS lock expiration time, this is different from veFXS.getBalance() which will return the locked amount of FXS after the lock has expired.”

This statement is in line with the code’s behavior. The _calculateVotingWeight function in the VeFxsVotingDelegation contract does not return the locked veFXS balance once a lock has expired.

If a voter’s lock has expired or was never created, the short-circuit condition returns zero voting power. This behavior contrasts with the veFxs.balanceOf function, which would return the user’s last locked FXS balance.

This divergence should be clearly documented in the code and should be reflected in Frax Finance’s public-facing documentation, which does not mention the fact that an expired lock does not hold any voting power: “Each veFXS has 1 vote in governance proposals. Staking 1 FXS for the maximum time, 4 years, would generate 4 veFXS. This veFXS balance itself will slowly decay down to 1 veFXS after 4 years, [...]”.

**Exploit Scenario**

Alice buys FXS to be able to vote on a proposal. She is not aware that she is required to create a lock (even if expired) to have any voting power at all. She is unable to vote for the proposal.

**Recommendations**

Short term, modify the VeFxsVotingDelegation contract to reflect the desired voting power curve and/or document whether this is intended behavior.

Long term, make sure to keep public-facing documentation up to date when changes are made.

## Upgradeable contract initialization calls are commented out

**Description**

The usage of OpenZeppelin’s upgradeable smart contract library requires that the parent contracts’ init functions are called in the child’s initialize function. However, this is commented out in the ShareableRebalancePool contract. Currently, these calls are no-ops and have no effect. However, prior to upgrading the library dependency, this code should be updated to reflect the target library’s implementation and ensure that the upgrade does not introduce a bug.

**Recommendations**

Short term, find an alternative solution to avoid hitting the maximum code size limit. Long term, avoid commented-out, dead code and ensure that library upgrades do not introduce new bugs.

## Project dependencies contain vulnerabilities

**Description**

Although dependency scans did not identify a direct threat to the project under review, npm and yarn audit identified dependencies with known vulnerabilities. Due to the sensitivity of the deployment code and its environment, it is important to ensure that dependencies are not malicious. Problems with dependencies in the JavaScript community could have a significant effect on the repository under review. The output below details these issues:

**Exploit Scenario** 

Alice installs the dependencies of the in-scope repository on a clean machine. Unbeknownst to Alice, a dependency of the project has become malicious or exploitable. Alice subsequently uses the dependency, disclosing sensitive information to an unknown actor.

**Recommendations**

Short term, ensure that the Shell Protocol dependencies are up to date. Several node modules have been documented as malicious because they execute malicious code when installing dependencies to projects. Keep modules current and verify their integrity after installation.

Long term, integrate automated dependency auditing into the development workflow. If a dependency cannot be updated when a vulnerability is disclosed, ensure that the code does not use and is not affected by the vulnerable functionality of the dependency.



