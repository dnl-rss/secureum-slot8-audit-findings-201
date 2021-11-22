# Consensys Findings

## Umbra audit

### 102. Umbra

**Finding**: Document potential edge cases for hook receiver contracts

**Description**: The functions `withdrawTokenAndCall()` and `withdrawTokenAndCallOnBehalf()` make a call to a hook contract designated by the owner of the withdrawing stealth address. There are very few constraints on the parameters to these calls in the `Umbra` contract itself. Anyone can force a call to a hook contract by transferring a small amount of tokens to an address that they control and withdrawing these tokens, passing the target address as the hook receiver.

**Recommendation**: Developers of these `UmbraHookReceiver` contracts should be sure to validate both the caller of the `tokensWithdrawn()` function and the function parameters.

### 103. Umbra

**Finding**: Document token behavior restrictions

**Description**: As with any protocol that interacts with arbitrary ERC20 tokens, it is important to clearly document which tokens are supported. Often this is best done by providing a specification for the behavior of the expected ERC20 tokens and only relaxing this specification after careful review of a particular class of tokens and their interactions with the protocol.

**Recommendation**: Known deviations from “normal” ERC20 behavior should be explicitly noted as NOT supported by the Umbra Protocol:
1. Deflationary or fee-on-transfer tokens:
    - These are tokens in which the balance of the recipient of a transfer may not be increased by the amount of the transfer.
    - There may also be some alternative mechanism by which balances are unexpectedly decreased.
    - While these tokens can be successfully sent via the `sendToken()` function, the internal accounting of the Umbra contract will be out of sync with the balance as recorded in the token contract, resulting in loss of funds.
2. Inflationary tokens:
    - The opposite of deflationary tokens.
    - The Umbra contract provides no mechanism for claiming positive balance adjustments.
3. Rebasing tokens:
    - A combination of the above cases, these are tokens in which an account’s balance increases or decreases along with expansions or contractions in supply.
    - The contract provides no mechanism to update its internal accounting in response to these unexpected balance adjustments, and funds may be lost as a result.

## DeFi DefiSaver Audit

### 104. DeFi Saver

**Finding**: Full test suite is recommended

**Description**: The test suite at this stage is not complete and many of the tests fail to execute. For complicated systems such as DeFi Saver, which uses many different modules and interacts with different DeFi protocols, it is crucial to have a full test coverage that includes the edge cases and failed scenarios. Especially this helps with safer future development and upgrading each module. As we’ve seen in some smart contract incidents, a complete test suite can prevent issues that might be hard to find with manual reviews.

**Recommendation**: Add a full coverage test suite.

### 105. DeFi Saver

**Finding**: Kyber `getRates` code is unclear

**Description**: Function names don’t reflect their true functionalities, and the code uses some undocumented assumptions.

**Recommendation**: Refactor the code to separate getting rate functionality with `getSellRate` and `getBuyRate`. Explicitly document any assumptions in the code ( slippage, etc).

### 106. DeFi Saver

**Finding**: Return value is not used for `TokenUtils.withdrawTokens`

**Description**: The return value of `TokenUtils.withdrawTokens`, which represents the actual amount of tokens that were transferred, is never used throughout the repository. This might cause discrepancy in the case where the original value of `_amount` was `type(uint256).max`.

**Recommendation**: The return value can be used to validate the withdrawal or used in the event emitted

### 107. DeFi Saver

**Finding**: Missing access control for `DefiSaverLogger.Log`

**Description**: `DefiSaverLogger` is used as a logging aggregator within the entire dapp, but anyone can create logs.

**Recommendation**: Add access control to all functions appropriately

## DAOfi Audit

### 108. DAOfi

**Finding**: Remove stale comments

**Description**: Remove inline comments that suggest the two `uint256` values `DAOfiV1Pair.reserveBase` and `DAOfiV1Pair.reserveQuote` are stored in the same storage slot. This is likely a carryover from the `UniswapV2Pair` contract, in which `reserve0`, `reserve1`, and `blockTimestampLast` are packed into a single storage slot.

**Recommendation**: Remove stale comments

## mstable-1.1 Audit

### 109. mstable-1.1

**Finding**: Discrepancy between code and comments

**Description**: There is a mismatch between what the code implements and what the corresponding comment describes that code implements.

**Recommendation**: Update the code or the comment to be consistent

## DAOfi Audit

### 110. DAOfi

**Finding**: Remove unnecessary call to `DAOfiV1Factory.formula()`:

**Description**: The `DAOfiV1Pair` functions `initialize()`, `getBaseOut()`, and `getQuoteOut()` all use the private function `_getFormula()`, which makes a call to the factory to retrieve the address of the `BancorFormula` contract. The formula address in the factory is set in the constructor and cannot be changed, so these calls can be replaced with an immutable value in the pair contract that is set in its constructor.

**Recommendation**: Remove unnecessary calls

### 111. DAOfi

**Finding**: Deeper validation of curve math

**Description**: Increased testing of edge cases in complex mathematical operations could have identified at least one issue raised in this report. Additional unit tests are recommended, as well as fuzzing or property-based testing of curve-related operations. Improperly validated interactions with the `BancorFormula` contract are seen to fail in unanticipated and potentially dangerous ways, so care should be taken to validate inputs and prevent pathological curve parameters.

**Recommendation**: More validation of mathematical operations

## Fei Protocol Audit

### 112. Fei Protocol

**Finding**: `GovernorAlpha` proposals may be canceled by the proposer, even after they have been accepted and queued

**Description**: `GovernorAlpha` allows proposals to be canceled via `cancel`. A proposer may cancel proposals in any of these states: `Pending`, `Active`, `Canceled`, `Defeated`, `Succeeded`, `Queued`, `Expired`.

**Recommendation**: Prevent proposals from being canceled unless they are in the `Pending` or `Active` states.

## eRLC Audit

### 113. eRLC

**Finding**: Require a delay period before granting `KYC_ADMIN_ROLE` Acknowledged:

**Description**: The KYC Admin has the ability to freeze the funds of any user at any time by revoking the `KYC_MEMBER_ROLE`. The trust requirements from users can be decreased slightly by implementing a delay on granting this ability to new addresses. While the management of private keys and admin access is outside the scope of this review, the addition of a time delay can also help protect the development team and the system itself in the event of private key compromise.

**Recommendation**: Use a `TimelockController` as the `KYC_DEFAULT_ADMIN` of the `eRLC` contract

## 1inch liquidity protocol audity

### 114. 1inch Liquidity Protocol

**Finding**: Improve inline documentation and test coverage

**Description**: The source-units hardly contain any inline documentation which makes it hard to reason about methods and how they are supposed to be used. Additionally, test-coverage seems to be limited. Especially for a public-facing exchange contract system test-coverage should be extensive, covering all methods and functions that can directly be accessed including potential security-relevant and edge-cases. This would have helped in detecting some of the findings raised with this report.

**Recommendation**: Consider adding natspec-format compliant inline code documentation, describe functions, what they are used for, and who is supposed to interact with them. Document function or source-unit specific assumptions. Increase test coverage.

### 115. 1inch Liquidity Protocol

**Finding**: Unspecific compiler version `pragma`

**Description**: For most source-units the compiler version `pragma` is very unspecific `^0.6.0`. While this often makes sense for libraries to allow them to be included with multiple different versions of an application, it may be a security risk for the actual application implementation itself. A known vulnerable compiler version may accidentally be selected or security tools might fall-back to an older compiler version ending up actually checking a different evm compilation that is ultimately deployed on the blockchain.

**Recommendation**: Avoid floating pragmas. We highly recommend pinning a concrete compiler version (latest without security issues) in at least the top-level “deployed” contracts to make it unambiguous which compiler version is being used. Rule of thumb: a flattened source-unit should have at least one non-floating concrete solidity compiler version pragma.

### 116. 1inch Liquidity Protocol

**Finding**: Use of hardcoded gas limits can be problematic

**Description**: Hardcoded gas limits can be problematic as the past has shown that gas economics in ethereum have changed, and may change again potentially rendering the contract system unusable in the future.

**Recommendation**: Be conscious about this potential limitation and prepare for the case where gas prices might change in a way that negatively affects the contract system.

### 117. 1inch Liquidity Protocol

**Finding**: Anyone can steal all the funds that belong to `ReferralFeeReceiver`

**Description**: The `ReferralFeeReceiver` receives pool shares when users `swap()` tokens in the pool. A `ReferralFeeReceiver` may be used with multiple pools and, therefore, be a lucrative target as it is holding pool shares. Any token or ETH that belongs to the `ReferralFeeReceiver` is at risk and can be drained by any user by providing a custom mooniswap pool contract that references existing token holdings. It should be noted that none of the functions in `ReferralFeeReceiver` verify that the user-provided mooniswap pool address was actually deployed by the linked `MooniswapFactory`.

**Recommendation**: Enforce that the user-provided mooniswap contract was actually deployed by the linked factory. Other contracts cannot be trusted. Consider implementing token sorting and de-duplication (`tokenA != tokenB`) in the pool contract constructor as well. Consider employing a reentrancy guard to safeguard the contract from reentrancy attacks. Improve testing. The methods mentioned here are not covered at all. Improve documentation and provide a specification that outlines how this contract is supposed to be used.

**Severity**: Critical

### 118. 1inch Liquidity Protocol

**Finding**: Unpredictable behavior for users due to admin front running or general bad timing

**Description**: In a number of cases, administrators of contracts can update or upgrade things in the system without warning. This has the potential to violate a security goal of the system. Specifically, privileged roles could use front running to make malicious changes just ahead of incoming transactions, or purely accidental negative effects could occur due to the unfortunate timing of changes. In general users of the system should have assurances about the behavior of the action they’re about to take.

**Recommendation**: We recommend giving the user advance notice of changes with a time lock. For example, make all system-parameter and upgrades require two steps with a mandatory time window between them. The first step merely broadcasts to users that a particular change is coming, and the second step commits that change after a suitable waiting period. This allows users that do not accept the change to withdraw immediately.

## Growth DeFi Audit

### 119. Growth DeFi

**Finding**: Improve system documentation and create a complete technical specification

**Description**: A system’s design specification and supporting documentation should be almost as important as the system’s implementation itself. Users rely on high-level documentation to understand the big picture of how a system works. Without spending time and effort to create palatable documentation, a user’s only resource is the code itself, something the vast majority of users cannot understand. Security assessments depend on a complete technical specification to understand the specifics of how a system works. When a behavior is not specified (or is specified incorrectly), security assessments must base their knowledge in assumptions, leading to less effective review. Maintaining and updating code relies on supporting documentation to know why the system is implemented in a specific way. If code maintainers cannot reference documentation, they must rely on memory or assistance to make high-quality changes. Currently, the only documentation for Growth DeFi is a single `README` file, as well as code comments.

**Recommendation**: Improve system documentation and create a complete technical specification

### 120. Growth DeFi

**Finding**: Ensure system states, roles, and permissions are sufficiently restrictive

**Description**: Smart contract code should strive to be strict. Strict code behaves predictably, is easier to maintain, and increases a system’s ability to handle nonideal conditions. Our assessment of Growth DeFi found that many of its states, roles, and permissions are loosely defined.

**Recommendation**: Document the use of administrator permissions. Monitor the usage of administrator permissions. Specify strict operation requirements for each contract.

### 121. Growth DeFi

**Finding**: Evaluate all tokens prior to inclusion in the system

**Description**: Review current and future tokens in the system for non-standard behavior. Particularly dangerous functionality to look for includes a callback (ie. `ERC777`) which would enable an attacker to execute potentially arbitrary code during the transaction, fees on transfers, or inflationary/deflationary tokens.

**Recommendation**: Evaluate all tokens prior to inclusion in the system

### 122. Growth DeFi

**Finding**: Use descriptive names for contracts and libraries

**Description**: The code base makes use of many different contracts, abstract contracts, interfaces, and libraries for inheritance and code reuse. In principle, this can be a good practice to avoid repeated use of similar code. However, with no descriptive naming conventions to signal which files would contain meaningful logic, codebase becomes difficult to navigate.

**Recommendation**: Use descriptive names for contracts and libraries

### 123. Growth DeFi

**Finding**: Prevent contracts from being used before they are entirely initialized

**Description**: Many contracts allow users to deposit / withdraw assets before the contracts are entirely initialized, or while they are in a semi-configured state. Because these contracts allow interaction on semi-configured states, the number of configurations possible when interacting with the system makes it incredibly difficult to determine whether the contracts behave as expected in every scenario, or even what behavior is expected in the first place.

**Recommendation**: Prevent contracts from being used before they are entirely initialized

### 124. Growth DeFi

**Finding**: Potential resource exhaustion by external calls performed within an unbounded loop

**Description**: `DydxFlashLoanAbstraction._requestFlashLoan` performs external calls in a potentially-unbounded loop. Depending on changes made to DyDx’s `SoloMargin`, this may render this flash loan provider prohibitively expensive. In the worst case, changes to `SoloMargin` could make it impossible to execute this code due to the block gas limit.

**Recommendation**: Reconsider or bound the loop

## Paxos Audit

### 125. Paxos

**Finding**: Owners can never be removed

**Description**: The intention of `setOwners()` is to replace the current set of owners with a new set of owners. However, the `isOwner` mapping is never updated, which means any address that was ever considered an owner is permanently considered an owner for purposes of signing transactions.

**Recommendation**: In `setOwners_()`, before adding new owners, loop through the current set of owners and clear their `isOwner` booleans

**Severity**: Critical

## Aave Protocol V2 Audit

### 126. Aave Protocol V2

**Finding**: Potential manipulation of stable interest rates using flash loans

**Description**: Flash loans allow users to borrow large amounts of liquidity from the protocol. It is possible to adjust the stable rate up or down by momentarily removing or adding large amounts of liquidity to reserves.

**Recommendation**: This type of manipulation is difficult to prevent especially when flash loans are available. Aave should monitor the protocol at all times to make sure that interest rates are being rebalanced to sane values.

## Aave Governance DAO Audit

### 127. Aave Governance DAO

**Finding**: Only whitelist validated assets

**Description**: Because some of the functionality relies on correct token behavior, any whitelisted token should be audited in the context of this system. Problems can arise if a malicious token is whitelisted because it can block people from voting with that specific token or gain unfair advantage if the balance can be manipulated.

**Recommendation**: Make sure to audit any new whitelisted asset.

## Aave CPM Price Provider Audit

### 128. Aave CPM Price Provider

**Finding**: Underflow if `TOKEN_DECIMALS` are greater than 18

**Description**: In `latestAnswer()`, the assumption is made that `TOKEN_DECIMALS` is less than 18.

**Recommendation**: Add a simple check to the `constructor` to ensure the added token has 18 decimals or less

### 129. Aave CPM Price Provider

**Finding**: Chainlink’s performance at times of price volatility

**Description**: In order to understand the risk of the Chainlink oracle deviating significantly, it would be helpful to compare historical prices on Chainlink when prices are moving rapidly, and see what the largest historical delta is compared to the live price on a large exchange.

**Recommendation**: Review Chainlink’s performance at times of price volatility

## Lien Protocol Audit

### 130. Lien Protocol

**Finding**: Consider an iterative approach to launching. Be aware of and prepare for worst-case scenarios

**Description**: The system has many components with complex functionality and no apparent upgrade path.

**Recommendation**: We recommend identifying which components are crucial for a minimum viable system, then focusing efforts on ensuring the security of those components first, and then moving on to the others. During the early life of the system, have a method for pausing and upgrading the system.

## Balancer Finance Audit

### 131. Balancer Finance

**Finding**: Use of modifiers for repeated checks

**Description**: It is recommended to use modifiers for common checks within different functions. This will result in less code duplication in the given smart contract and adds significant readability into the code base.

**Recommendation**: Use of modifiers for repeated checks

### 132. Balancer Finance

**Finding**: Switch modifier order

**Description**: `BPool` functions often use modifiers in the following order: `_logs_`, `_lock_`. Because `_lock_` is a reentrancy guard, it should take precedence over `_logs_`.

**Recommendation**: Place `_lock_` before other modifiers; ensuring it is the very first and very last thing to run when a function is called.

## MCDEX Mai Protocol V2 Audit

### 133. MCDEX Mai Protocol V2

**Finding**: Address codebase fragility

**Description**: Software is considered “fragile” when issues or changes in one part of the system can have side-effects in conceptually unrelated parts of the codebase. Fragile software tends to break easily and may be challenging to maintain.

**Recommendation**: Building an anti-fragile system requires careful thought and consideration outside of the scope of this review. In general, prioritize the following concepts:
1. Follow the single-responsibility principle of functions
2. Reduce reliance on external systems
