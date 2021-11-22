# Trail of Bits Audit findings

## Liquidity Audit

### 134. Liquidity

**Finding**: Reentrancy could lead to incorrect order of emitted events

**Description**: The order of operations in the `_moveTokensAndETHfromAdjustment` function in the `BorrowOperations` contract may allow an attacker to cause events to be emitted out of order. In the event that the borrower is a contract, this could trigger a callback into `BorrowerOperations`, executing the `_adjustTrove` flow above again. As the `_moveTokensAndETHfromAdjustment` call is the final operation in the function the state of the system on-chain cannot be manipulated. However, there are events that are emitted after this call. In the event of a reentrant call, these events would be emitted in the incorrect order. The event for the second operation is emitted first, followed by the event for the first operation. Any off-chain monitoring tools may now have an inconsistent view of on-chain state.

**Recommendation**: Apply the checks-effects-interactions pattern and move the event emissions above the call to `_moveTokensAndETHfromAdjustment` to avoid the potential reentrancy.

## Origin Dollar Audit

### 135. Origin Dollar

**Finding**: Variable shadowing from OUSD to ERC20

**Description**: OUSD inherits from ERC20, but redefines the `_allowances` and `_totalSupply` state variables. As a result, access to these variables can lead to returning different values.

**Recommendation**: Remove the shadowed variables (`_allowances` and `_totalSupply`) in OUSD.

### 136. Origin Dollar

**Finding**: `VaultCore.rebase` functions have no return statements

**Description**: `VaultCore.rebase()` and `VaultCore.rebase(bool)` return a `uint` but lack a return statement. As a result these functions will always return the default value, and are likely to cause issues for their callers. Both `VaultCore.rebase()` and `VaultCore.rebase(bool)` are expected to return a `uint256`. `rebase()` does not have a return statement. `rebase(bool)` has one return statement in one branch (return 0), but lacks a return statement for the other paths. So both functions will always return zero. As a result, a third-party code relying on the return value might not work as intended.

**Recommendation**: Add the missing return statement(s) or remove the return type in `VaultCore.rebase()` and `VaultCore.rebase(bool)`. Properly adjust the documentation as necessary.

### 137. Origin Dollar

**Finding**: Multiple contracts are missing inheritances

**Description**: Multiple contracts are the implementation of their interfaces, but do not inherit from them. This behavior is error-prone and might lead the implementation to not follow the interface if the code is updated.

**Recommendation**: Ensure contracts inherit from their interfaces

## Yield Protocol Audit

### 138. Yield Protocol

**Finding**: Solidity compiler optimizations can be dangerous

**Description**: Yield Protocol has enabled optional compiler optimizations in Solidity. There have been several bugs with security implications related to optimizations. Moreover, optimizations are actively being developed. Solidity compiler optimizations are disabled by default, and it is unclear how many contracts in the wild actually use them. Therefore, it is unclear how well they are being tested and exercised. High-severity security issues due to optimization bugs have occurred in the past. A high-severity bug in the emscripten-generated solc-js compiler used by Truffle and Remix persisted until late 2018. The fix for this bug was not reported in the Solidity CHANGELOG. Another high-severity optimization bug resulting in incorrect bit shift results was patched in Solidity 0.5.6 .

**Recommendation**:
- Short term, measure the gas savings from optimizations, and carefully weigh them against the possibility of an optimization-related bug.
- Long term, monitor the development and adoption of Solidity compiler optimizations to assess their maturity.

### 139. Yield Protocol

**Finding**: Permission-granting is too simplistic and not flexible enough

**Description**: The Yield Protocol contracts implement an oversimplified permission system that can be abused by the administrator. The Yield Protocol implements several contracts that need to call privileged functions from each other. However, there is no way to specify which operation can be called for every privileged user. All the authorized addresses can call any restricted function, and the owner can add any number of them. Also, the privileged addresses are supposed to be smart contracts; however, there is no check for that. Moreover, once an address is added, it cannot be deleted.

**Recommendation**: Rewrite the authorization system to allow only certain addresses to access certain functions

### 140. Yield Protocol

**Finding**: Lack of validation when setting the maturity value

**Description**: When a fyDAI contract is deployed, one of the deployment parameters is a maturity date, passed as a Unix timestamp. This is the date at which point fyDAI tokens can be redeemed for the underlying Dai. Currently, the contract constructor performs no validation on this timestamp to ensure it is within an acceptable range. As a result, it is possible to mistakenly deploy a YDai contract that has a maturity date in the past or many years in the future, which may not be immediately noticed.

**Recommendation**:
- Short term, add checks to the YDai contract constructor to ensure maturity timestamps fall within an acceptable range. This will prevent maturity dates from being mistakenly set in the past or too far in the future.
- Long term, always perform validation of parameters passed to contract constructors. This will help detect and prevent errors during deployment.

### 141. Yield Protocol

**Finding**: Delegates can be added or removed repeatedly to bloat logs

**Description**: Several contracts in the Yield Protocol system inherit the Delegable contract. This contract allows users to delegate the ability to perform certain operations on their behalf to other addresses. When a user adds or removes a delegate, a corresponding event is emitted to log this operation. However, there is no check to prevent a user from repeatedly adding or removing a delegation that is already enabled or revoked, which could allow redundant events to be emitted repeatedly.

**Recommendation**:
- Short term, add a require statement to check that the delegate address is not already enabled or disabled for the user. This will ensure log messages are only emitted when a delegate is activated or deactivated.
- Long term, review all operations and avoid emitting events in repeated calls to idempotent operations. This will help prevent bloated logs.

## 0x Protocol Audit

### 142. 0x Protocol

**Finding**: Lack of events for critical operations

**Description**: Several critical operations do not trigger events, which will make it difficult to review the correct behavior of the contracts once deployed. Users and blockchain monitoring systems will not be able to easily detect suspicious behaviors without events.

**Recommendation**:
- Short term, add events where appropriate for all critical operations.
- Long term, consider using a blockchain monitoring system to track any suspicious behavior in the contracts.

### 143. 0x Protocol

**Finding**: `_assertStakingPoolExists` never returns true

**Description**: The `_assertStakingPoolExists` should return a `bool` to determine if the staking pool exists or not; however, it only returns `false` or reverts. The `_assertStakingPoolExists` function checks that a staking pool exists given a pool id parameter. However, this function does not use a `return` statement and therefore, it will always return `false` or revert.

**Recommendation**: Add a `return` statement or remove the return type. Properly adjust the documentation, if needed.

## DFX Finance Audit

### 144. DFX Finance

**Finding**: `_min` and `_max` have unorthodox semantics

**Description**: Throughout the Curve contract, `_minTargetAmount` and `_maxOriginAmount` are used as open ranges (i.e., ranges that exclude the value itself). This contravenes the standard meanings of the terms “minimum” and “maximum,” which are generally used to describe closed ranges.

**Recommendation**:
- Short term, unless they are intended to be strict, make the inequalities in the require statements non-strict. Alternatively, consider refactoring the variables or providing additional documentation to convey that they are meant to be exclusive bounds.
- Long term, ensure that mathematical terms such as “minimum,” “at least,” and “at most” are used in the typical way—that is, to describe values inclusive of minimums or maximums (as relevant).

### 145. DFX Finance

**Finding**: `CurveFactory.newCurve` returns existing curves without provided arguments

**Description**: `CurveFactory.newCurve` takes values and creates a Curve contract instance for each `_baseCurrency` and `_quoteCurrency` pair, populating the Curve with provided weights and assimilator contract references. However, if the pair already exists, the existing Curve will be returned without any indication that it is not a newly created Curve with the provided weights. If an operator attempts to create a new Curve for a base-and-quote-currency pair that already exists, `CurveFactory` will return the existing Curve instance regardless of whether other creation parameters differ. A naive operator may overlook this issue.

**Recommendation**: Consider rewriting `newCurve` such that it reverts in the event that a base-and-quote-currency pair already exists. A view function can be used to check for and retrieve existing Curves without any gas payment prior to an attempt at Curve creation.

### 146. DFX Finance

**Finding**: Missing zero-address checks in `Curve.transferOwnership` and `Router.constructor`

**Description**: Like other similar functions, `Curve._transfer` and `Orchestrator.includeAsset` perform zero-address checks. However, `Curve.transferOwnership` and the `Router` constructor do not. This may make sense for `Curve.transferOwnership`, because without zero-address checks, the function may serve as a means of burning ownership. However, popular contracts that define similar functions often consider this case, such as OpenZeppelin’s `Ownable` contracts. Conversely, a zero-address check should be added to the `Router` constructor to prevent the deployment of an invalid `Router`, which would revert upon a call to the zero address.

**Recommendation**:
- Short term, consider adding zero-address checks to the Router’s constructor and Curve’s `transferOwnership` function to prevent operator errors.
- Long term, review state variables which referencing contracts to ensure that the code that sets the state variables performs zero-address checks where necessary

### 147. DFX Finance

**Finding**: `safeApprove` does not check return values for approve call

**Description**: Although the `Router` contract uses OpenZeppelin’s SafeERC20 library to perform safe calls to ERC20’s `approve` function, the `Orchestrator` library defines its own `safeApprove` function. This function checks that a call to `approve` was successful but does not check `returndata` to verify whether the call returned `true`. In contrast, OpenZeppelin’s `safeApprove` function checks return values appropriately. This issue may result in uncaught approve errors in successful Curve deployments, causing undefined behavior.

**Recommendation**:
- Short term, leverage OpenZeppelin’s `safeApprove` function wherever possible.
- Long term, ensure that all low-level calls have accompanying contract existence checks and return value checks where appropriate.

### 148. DFX Finance

**Finding**: ERC20 token Curve does not implement `symbol`, `name`, or `decimals`

**Description**: `Curve.sol` is an ERC20 token and implements all six required ERC20 methods: `balanceOf`, `totalSupply`, `allowance`, `transfer`, `approve`, and `transferFrom`. However, it does not implement the optional but extremely common view methods `symbol`, `name`, and `decimals`.

**Recommendation**:
- Short term, implement `symbol`, `name`, and `decimals` on Curve contracts.
- Long term, ensure that contracts conform to all required and recommended industry standards.

### 149. DFX Finance

**Finding**: Insufficient use of SafeMath

**Description**: `CurveMath.calculateTrade` is used to compute the output amount for a trade. However, although `SafeMath` is used throughout the codebase to prevent underflows/overflows, it is not used in this calculation. Although we could not prove that the lack of `SafeMath` would cause an arithmetic issue in practice, all such calculations would benefit from the use of `SafeMath`.

**Recommendation**: Review all critical arithmetic to ensure that it accounts for underflows, overflows, and the loss of precision. Consider using `SafeMath` and the safe functions of `ABDKMath64x64` where possible to prevent underflows and overflows.

### 150. DFX Finance

**Finding**: `setFrozen` can be front-run to deny deposits/swaps

**Description**: Currently, a `Curve` contract owner can use the `setFrozen` function to set the contract into a state that will block swaps and deposits. A contract owner could leverage this process to front-run transactions and freeze contracts before certain deposits or swaps are made; the contract owner could then unfreeze them at a later time.

**Recommendation**:
- Short term, consider rewriting `setFrozen` such that any contract freeze will not last long enough for a malicious user to easily execute an attack. Alternatively, depending on the intended use of this function, consider implementing permanent freezes.

## Hermez Network Audit

### 151. Hermez Network

**Finding**: Account creation spam

**Description**: `Hermez` has a limit of `2**MAX_NLEVELS` accounts. There is no fee on account creation, so an attacker can spam the network with account creation to fill the tree. If `MAX_NLEVELS` is below 32, an attacker can quickly reach the account limit. If `MAX_NLEVELS` is above or equal to 32, the time required to fill the tree will depend on the number of transactions accepted per second, but will take at least a couple of months. Ethereum miners do not have to pay for account creation. Therefore, an Ethereum miner can spam the network with account creation by sending L1 user transactions.

**Recommendation**:
- Short term, add a fee for account creation or ensure `MAX_NLEVELS` is at least 32. Also, monitor account creation and alert the community if a malicious coordinator spams the system. This will prevent an attacker from spamming the system to prevent new accounts from being created.
- Long term, when designing spam mitigation, consider that L1 gas cost can be avoided by Ethereum miners.

### 152. Hermez Network

**Finding**: Using empty functions instead of interfaces leaves contract error-prone

**Description**: `WithdrawalDelayerInterface` is a contract meant to be an interface. It contains functions with empty bodies instead of function signatures, which might lead to unexpected behavior. A contract inheriting from `WithdrawalDelayerInterface` will not require an override of these functions and will not benefit from the compiler checks on its correct interface.

**Recommendation**:
- Short term, use an interface instead of a contract in `WithdrawalDelayerInterface`. This will make derived contracts follow the interface properly.
- Long term, properly document the inheritance schema of the contracts. Use Slither’s `inheritance-graph` printer to review the inheritance.

### 153. Hermez Network

**Finding**: `cancelTransaction` can be called on non-queued transaction

**Description**: Without a transaction existence check in `cancelTransaction`, an attacker can confuse monitoring systems. `cancelTransaction` emits an event without checking that the transaction to be canceled exists. This allows a malicious admin to confuse monitoring systems by generating malicious events.

**Recommendation**:
- Short term, check that the transaction to be canceled exists in `cancelTransaction`. This will ensure that monitoring tools can rely on emitted events.
- Long term, write a specification of each function and thoroughly test it with unit tests and fuzzing. Use symbolic execution for arithmetic invariants.

### 154. Hermez Network

**Finding**: Contracts used as dependencies do not track upstream changes

**Description**: Third-party contracts like `_concatStorage` are pasted into the `Hermez` repository. Moreover, the code documentation does not specify the exact revision used, or if it is modified. This makes updates and security fixes on these dependencies unreliable since they must be updated manually. `_concatStorage` is borrowed from the `solidity-bytes-utils` library, which provides helper functions for byte-related operations. Recently, a critical vulnerability was discovered in the library’s slice function which allows arbitrary writes for user-supplied inputs.

**Recommendation**:
- Short term, review the codebase and document each dependency’s source and version. Include the third-party sources as submodules in your Git repository so internal path consistency can be maintained and dependencies are updated periodically.
- Long term, identify the areas in the code that are relying on external libraries and use an Ethereum development environment and NPM to manage packages as part of your project.

### 155. Hermez Network

**Finding**: Expected behavior regarding authorization for adding tokens is unclear

**Description**: `addToken` allows anyone to list a new token on `Hermez`. This contradicts the online documentation, which implies that only the governance should have this authorization. It is unclear whether the implementation or the documentation is correct.

**Recommendation**:
- Short term, update either the implementation or the documentation to standardize the authorization specification for adding tokens.
- Long term, write a specification of each function and thoroughly test it with unit tests and fuzzing. Use symbolic execution for arithmetic invariants.

### 156. Hermez Network

**Finding**: Contract name duplication leaves codebase error-prone

**Description**: The codebase has multiple contracts that share the same name. This allows `buidler-waffle` to generate incorrect json artifacts, preventing third parties from using their tools. `buidler-waffle` does not correctly support a codebase with duplicate contract names. The compilation overwrites compilation artifacts and prevents the use of third-party tools, such as `slither`.

**Recommendation**:
- Short term, prevent the re-use of duplicate contract names or change the compilation framework.
- Long term, use Slither, which will help detect duplicate contract names.

## Advanced Blockchains Audit

### 157. Advanced Blockchains

**Finding**: Use of hard-coded addresses may cause errors

**Description**: Each contract needs contract addresses in order to be integrated into other protocols and systems. These addresses are currently hard-coded, which may cause errors and result in the codebase’s deployment with an incorrect asset. Using hard-coded values instead of deployer-provided values makes these contracts incredibly difficult to test.

**Recommendation**:
- Short term, set addresses when contracts are created rather than using hard-coded values. This practice will facilitate testing.
- Long term, to ensure that contracts can be tested and reused across networks, avoid using hard-coded parameters.

### 158. Advanced Blockchains

**Finding**: Borrow rate depends on approximation of blocks per year

**Description**: The borrow rate formula uses an approximation of the number of blocks mined annually. This number can change across different blockchains and years. The current value assumes that a new block is mined every 15 seconds, but on Ethereum mainnet, a new block is mined every ~13 seconds. To calculate the base rate, the formula determines the approximate borrow rate over the past year and divides that number by the estimated number of blocks mined per year. However, `blocksPerYear` is an estimated value and may change depending on transaction throughput. Additionally, different blockchains may have different block-settling times, which could also alter this number.

**Recommendation**:
- Short term, analyze the effects of a deviation from the actual number of blocks mined annually in borrow rate calculations and document the associated risks.
- Long term, identify all variables that are affected by external factors, and document the risks associated with deviations from their true values.

### 159. Advanced Blockchains

**Finding**: Flash loan rate lacks bounds and can be set arbitrarily

**Description**: There are no lower or upper bounds on the flash loan rate implemented in the contract. The Blacksmith team could therefore set an arbitrarily high flash loan rate to secure higher fees. The Blacksmith team sets the `_flashLoanRate` when the `Vault` is first initialized. The `blackSmithTeam` address can then update this value by calling `updateFlashloanRate`. However, because there is no check on either setter function, the flash loan rate can be set arbitrarily. A very high rate could enable the Blacksmith team to steal vault deposits.

**Recommendation**:
- Short term, introduce lower and upper bounds for all configurable parameters in the system to limit privileged users’ abilities.
- Long term, identify all incoming parameters in the system as well as the financial implications of large and small corner-case values. Additionally, use Echidna or Manticore to ensure that system invariants hold.

### 160. Advanced Blockchains

**Finding**: Logic duplicated across code

**Description**: The logic in the repositories provided to Trail of Bits contains a significant amount of duplicated code. This development practice increases the risk that new bugs will be introduced into the system, as bug fixes must be copied and pasted into files across the system.

**Recommendation**:
- Short term, use inheritance to allow code to be reused across contracts. Changes to one inherited contract will be applied to all files without requiring developers to copy and paste them.
- Long term, minimize the amount of manual copying and pasting required to apply changes made to one file to other files.

### 161. Advanced Blockchains

**Finding**: Insufficient testing

**Description**: The repositories under review lack appropriate testing, which increases the likelihood of errors in the development process and makes the code more difficult to review.

**Recommendation**:
- Short term, ensure that the unit tests cover all public functions at least once, as well as all known corner cases.
- Long term, integrate coverage analysis tools into the development process and regularly review the coverage.

### 162. Advanced Blockchains

**Finding**: Project dependencies contain vulnerabilities

**Description**: Although dependency scans did not yield a direct threat to the projects under review, `yarn audit` identified dependencies with known vulnerabilities. Due to the sensitivity of the deployment code and its environment, it is important to ensure dependencies are not malicious. Problems with dependencies in the JavaScript community could have a significant effect on the repositories under review.

**Recommendation**:
- Short term, ensure dependencies are up to date. Several node modules have been documented as malicious because they execute malicious code when installing dependencies to projects. Keep modules current and verify their integrity after installation.
- Long term, consider integrating automated dependency auditing into the development workflow. If dependencies cannot be updated when a vulnerability is disclosed, ensure that the codebase does not use and is not affected by the vulnerable functionality of the dependency.

### 163. Advanced Blockchains

**Finding**: Lack of contract documentation makes codebase difficult to understand

**Description**: The codebase lacks code documentation, high-level descriptions, and examples, making the contracts difficult to review and increasing the likelihood of user mistakes. The documentation would benefit from more detail.

**Recommendation**:
- Short term, review and properly document the above mentioned aspects of the codebase.
- Long term, consider writing a formal specification of the protocol.

### 164. Advanced Blockchains

**Finding**: `ABIEncoderV2` is not production-ready

**Description**: The contracts use the new Solidity ABI encoder, `ABIEncoderV2`. This experimental encoder is not ready for production. More than 3% of all GitHub issues for the Solidity compiler are related to experimental features, primarily `ABIEncoderV2`. Several issues and bug reports are still open and unresolved. `ABIEncoderV2` has been associated with more than 20 high-severity bugs, some of which are so recent that they have not yet been included in a Solidity release. For example, in March 2019 a severe bug introduced in Solidity 0.5.5 was found in the encoder.

**Recommendation**:
- Short term, use neither `ABIEncoderV2` nor any other experimental Solidity feature. Refactor the code such that structs do not need to be passed to or returned from functions.
- Long term, integrate static analysis tools like `Slither` into your CI pipeline to detect unsafe pragmas.

## dForce Lending Audit

### 165. dForce Lending

**Finding**: Contract owner has too many privileges

**Description**: The owner of the contracts has too many privileges relative to standard users. Users can lose all of their assets if a contract owner private key is compromised. The contract owner can do the following:
1. Upgrade the system’s implementation to steal funds
2. Upgrade the token’s implementation to act maliciously
3.  Increase the amount of iTokens for reward distribution to such an extent that rewards cannot be disbursed
4. Arbitrarily update the interest model contracts The concentration of these privileges creates a single point of failure. It increases the likelihood that the owner will be targeted by an attacker, especially given the insufficient protection on sensitive owner private keys. Additionally, it incentivizes the owner to act maliciously.

**Recommendation**:
- Short term:
    1. Clearly document the functions and implementations the owner can change.
    2. Split privileges to ensure that no one address has excessive ownership of the system.
- Long term, document the risks associated with privileged users and single points of failure. Ensure that users are aware of all the risks associated with the system.

### 166. dForce Lending

**Finding**: Poor error-handling practices in test suite

**Description**: The test suite does not properly test expected behavior, as the contracts run in production. Additionally, certain components lack error-handling methods. These deficiencies can cause failed tests to be overlooked. In particular, the tests fail to properly check error messages. For example, errors are silenced with a try-catch statement. If this error is silenced, there will be no guarantee that a smart contract call has reverted for the right reason. As a result, if the test suite passes, it will provide no guarantee that the transaction call reverted correctly.

**Recommendation**:
- Short term, test these operations against a specific error message. Testing will ensure that errors are never silenced, and the test suite will check that a contract call has reverted for the right reason.
- Long term, follow standard testing practices for smart contracts to minimize the number of issues during development.
