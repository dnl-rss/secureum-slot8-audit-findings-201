# Sigma Prime Audit Findings

## Ether Collateral Audit

### 167. Synthetix EtherCollateral

**Finding**: Redundant and Unused Code

**Description**: The `_recordLoanClosure()` function returns a boolean ( `loanClosed` ) which is never used by the calling function (see `_closeLoan()` , line [312]). Furthermore, since the `_recordLoanClosure()` function is only called via the `_closeLoan()` function, this means that `synthLoan.timeClosed` is always equal to zero (see require statement on line [305]). Therefore, the if statement on line [357] is redundant and unnecessary.

**Recommendation**:
1. Using the return value of the `_recordLoanClosure()` function or changing the function definition to stop returning `loanClosed`
2. Removing the `if` statement in line [357]

### 168.Synthetix EtherCollateral

**Finding**: Single Account Can Capture All Supply

**Description**: The `EtherCollateral` smart contract does not rely on a `maxLoanSize` to limit the amount of ETH that can be locked for a loan. As a result, a single account can issue a loan that will reach the total minting supply.

**Recommendation**: Make sure this behaviour is understood and consider introducing and enforcing a cap ( `maxLoanSize` ) on the size of the loans allowed to be opened.

### 169.Synthetix EtherCollateral

**Finding**: Insufficient Input Validation

**Description**: The constructor of the `EtherCollateral` smart contract does not check the validity of the addresses provided as input parameters. It is possible to deploy an instance of the `EtherCollateral` contract with the `synthProxy`, `sUSDProxy`, and depot addresses set to zero. Similarly, the effective interest rate can be equal to zero if `interestRate` is set to any value lesser than 31536000 ( `SECONDS_IN_A_YEAR` ), as `interestPerSecond` will be null.

**Recommendation**: Consider introducing `require` statements to perform adequate input validation.

### 170. Synthetix EtherCollateral

**Finding**: Unused Event Logs

**Description**: log events are declared but never emitted.

**Recommendation**: Remove these events from the `EtherCollateral` contract.

## InfiniGold Audit

### 171. InfiniGold

**Finding**: Possible Unintended Token Burning in `transferFrom()` Function:

**Description**: `InfiniGold` allows users to convert/exchange their PMGT tokens to "gold certificates", which are digital artefacts effectively redeemable for actual gold. To do so, users are supposed to send their PMGT tokens to a specific burn address. The `transferFrom()` function does not check the to address against this burn address. Users may send tokens to the burn address, using the `transferFrom()` function, without triggering the emission of the `Burn(address indexed burner, uint256 value)` event, which dictates how the gold certificates are created and distributed.

**Recommendation**: Prevent sending tokens to the burn address in the `transferFrom()` function. This can be achieved by adding a require within `transferFrom()` which disallows the to address to be the `burnAddress`.

### 172. InfiniGold

**Finding**: Denial of Service Vector from Unbound List

**Description**: The `reset()` internal function (called by the `replaceAll()` function) resets the role linked list by deleting all the elements (i.e. nodes) part of the bearer mapping. The caller is bound by the number of elements that are being removed for a particular role. Calling the `reset()` function will exceed the current block gas limit (i.e. 8,000,0000) for more than 371 total elements in a role linked list. Similarly, the `size()` and `toArray()` functions also loop through the linked list. This essentially means that listers, unlisters, minters, pausers, unpausers and owners can perform denial of service attacks on the lists they administer. In a scenario where the `Roles` library is leveraged by other smart contracts, calling these two functions will also result in a potential denial of service after a certain number of elements have been included in the linked list (this number would depend on the gas cost of the Opcodes implemented by the calling functions).

**Recommendation**: One way to ensure that the current block gas limit is not exceeded would be to introduce a condition in the `add()` function to check that the linked list size is strictly lesser than 371 elements before adding a new element. This additional condition would significantly increase the gas cost associated with calling the `add()` function, as a call to the `size()` function would be required to fetch the exact number of nodes in the linked list. Alternatively, the `gasleft()` Solidity special function could be used to make sure that going through the linked list does not exceed the block gas limit. Finally, the `reset()` could be changed to allow for removing an arbitrary number of nodes (by taking this number as a function parameter).

### 173. InfiniGold

**Finding**: ERC20 Implementation Vulnerable to Front-Running

**Description**: Front-running attacks involve users watching the blockchain for particular transactions and, upon observing such a transaction, submitting their own transactions with a greater gas price. This incentivises miners to prioritize the later transaction. The ERC20 implementation is known to be affected by a front-running vulnerability, in its `approve()` function.

**Recommendation**: Be aware of the front-running issues in `approve()`, potentially add extended approve functions which are not vulnerable to the front-running vulnerability for future third-party-applications. See the Open-Zeppelin [8] solution for an example. We note that modifying the ERC20 standard to address this issue may lead to backward incompatibilities with external third-party software.

### 174. InfiniGold

**Finding**: Unnecessary `require` Statement

**Description**: The following require statement in `Blacklistable.sol` can be removed: `require(to != address(0))`; Indeed, this check is implemented in the `_transfer()` function in the `ERC20.sol` smart contract.

**Recommendation**: Consider removing the require statement for gas saving purposes.

## Synthetix Unipool Audit

### 175. Synthetix Unipool

**Finding**: Rounding to Zero if Duration is Greater Than Reward

**Description**: The `rewardRate` value is calculated as follows: `rewardRate = reward/duration`. Due to the integer representation of these variables, if duration is larger than reward the value of `rewardRate` will round to zero. Thus, stakers will not receive any of the reward for their stakes. Furthermore, due to the integer rounding, the total rewards distributed may be rounded down by up to one less than duration. As a result, the `Unipool` contract may slowly accumulate SNX.

**Recommendation**: Beware of the rounding issues when calling the `notifyRewardAmount()` function. We also recommend some way of allowing the excess SNX reward from rounding to be claimed or withdrawn from the Unipool contract.

### 176. Synthetix Unipool

**Finding**: Withdrawn Event Log Poisoning

**Description**: Calling the `withdraw()` function will `emit` the `Withdrawn` event. No UNI tokens are required as this function can be called with `amount = 0`. As a result a user could continually call this function, creating a potentially infinite amount of events. This can lead to an event log poisoning situation where malicious external users spam the `Unipool` contract to generate arbitrary `Withdrawn` events.

**Recommendation**: Consider adding a `require` or `if` statement preventing the `withdraw()` function from emitting the `Withdrawn` event when the `amount` variable is zero.
