# QA report and Non Critical

## 1. Event is missing `indexed` fields.

_Index event fields make the field more quickly accessible to off-chain tools that parse events. However, note that each index field costs extra gas during emission, so it’s not necessarily best to index the maximum allowed per event (threefields). Each event should use three indexed fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. `If there are fewer than three fields, all of the fields should be indexed`._

_Each `event` should use three `indexed` fields if there are three or more fields_

Note: Using the `indexed` keyword for value types such as `uint`, `bool`, and `address` saves gas costs, as seen in the example below. However, this is only the case for value types, whereas indexing `bytes` and `strings` are more expensive than their unindexed version.

Before:

```java
event Withdraw(uint256, address);

function withdraw(uint256 amount) public {
    emit Withdraw(amount, msg.sender);
}
```

After:

```java
event Withdraw(uint256 indexed, address indexed);

function withdraw(uint256 amount) public {
    emit Withdraw(amount, msg.sender);
}
```

<br>
<hr>

## 2. Remove `include` for hardhat’s console

```java
import 'hardhat/console.sol';
```

<br>
<hr>

## 3. Numeric values having to do with time should use time units for readability

_There are [units](https://docs.soliditylang.org/en/latest/units-and-global-variables.html#time-units) for seconds, minutes, hours, days, and weeks, and since they’re defined, they should be used_

Example:

```java
/// @audit 600000
48:       uint256 constant dailyEmission = 600000 * 10**18;
/// @audit 60
/// @audit 60
57:       uint256 constant secsInDay = 24 * 60 * 60;
```

```java
/// @audit 86400
296:      uint256 internal constant MAXTIME = 4 * 365 * 86400;
/// @audit 86400
297:      int128 internal constant iMAXTIME = 4 * 365 * 86400;
```

<br>
<hr>

## 4. Use scientific notation (e.g. `1e18`) rather than exponentiation (e.g. `10**18`)

_While the compiler knows to optimize away the exponentiation, it’s still better coding practice to use idioms that do not require compiler optimization, if they exist._

Example:

```java
48:       uint256 constant dailyEmission = 600000 * 10**18;
100:          if (rewardToken.totalSupply() > 1000000000 * 10**18) {...}
```

<br>

## 4.1 Large multiples of ten should use scientific notation (e.g. `1e6`) rather than decimal literals (e.g. `1000000`), for readability

Example:

```java
48:       uint256 constant dailyEmission = 600000 * 10**18;
100:          if (rewardToken.totalSupply() > 1000000000 * 10**18) {...}
308:      mapping(uint256 => Point[1000000000]) public user_point_history; // user -> Point[user_epoch]
```

<br>

## 4.2 Or use `_` to make long numeric literals easier to read

```java
uint256 x = 1000000000000; // A trillion
uint256 y = 1_000_000_000_000; // Also a trillion but a tad more readable
```

<br>
<hr>

## 5. Variable names that consist of all capital letters should be reserved for `constant`/`immutable` variables.

Example:

```java
uint256 public MIN_VOTING_POWER_REQUIRED = 0;
```

<br>
<hr>

## 6. Adding a `return` statement when the function defines a named return variable, is redundant.

Example:

```java
 function getIndexFromElement(uint256 uid, uint256[] storage array)
        internal
        view
        returns (uint256 _index)
    {
        bool assigned = false;
        for (uint256 index = 0; index < array.length; index++) {
            if (uid == array[index]) {
                _index = index;
                assigned = true;
                return _index;
            }
        }
        require(assigned, "Didnt Find that element in live list, cannot scrub");
    }
```

<br>
<hr>

## 7. `type(uint<n>).max` should be used instead of `uint<n>(-1)`

_`type(X)` returns information about `X` type. [Doc](https://docs.soliditylang.org/en/latest/units-and-global-variables.html#type-information)_

The following properties are available for an integer type T:

-   `type(T).min` The smallest value representable by type T.
-   `type(T).max` The largest value representable by type T.

Example:

```java
 if (allowance[from][msg.sender] != uint256(-1)) {...}
```

<br>
<hr>

## 8. `ecrecover()` signature validity not checked

_`ecrecover()` returns the zero address if the signature is invalid. If the signer provided is also zero, then all incorrect signatures will be allowed_

```java
176          address signaturesigner = ecrecover(hash, o.v, o.r, o.s);
177          require(signaturesigner == o.signer, 'invalid signature');
178          if (signaturesigner != o.signer) {
179              return (0, hashStruct, 0);
180:         }
```

<br>
<hr>

## 9. Consider addings checks for signature malleability - Signatures vulnerable to malleability attacks.

_Use OpenZeppelin's `ECDSA` contract rather than calling `ecrecover()` directly_

_`ecrecover()` accepts as valid, two versions of signatures, meaning an attacker can use the same signature twice. Consider adding checks for signature malleability, or using OpenZeppelin’s `ECDSA` library to perform the extra checks necessary in order to prevent this attack._

```java
  address recoveredAddress = ecrecover(digest, v, r, s);
```

<br>
<hr>

## 10. Don't use of `Block.Timestamp` can be manipulated

[Docs](https://solidity-by-example.org/hacks/block-timestamp-manipulation/)
_Block timestamps have historically been used for a variety of applications, such as entropy for random numbers (see the Entropy Illusion for further details), locking funds for periods of time, and various state-changing conditional statements that are time-dependent. Miners have the ability to adjust timestamps slightly, which can prove to be dangerous if block timestamps are used incorrectly in smart contracts._

Example:

```java
src/Swap/BaseV1-core.sol:96:        observations.push(Observation(block.timestamp, 0, 0,0));
src/Swap/BaseV1-core.sol:138:       uint blockTimestamp = block.timestamp;
src/Swap/BaseV1-core.sol:159:       blockTimestamp = block.timestamp;
src/Swap/BaseV1-core.sol:176:       if (block.timestamp == _observation.timestamp) {
src/Swap/BaseV1-core.sol:180:       uint timeElapsed = block.timestamp - _observation.timestamp;
src/Swap/BaseV1-core.sol:471:       require(deadline >= block.timestamp, "BaseV1: EXPIRED");
src/Swap/BaseV1-periphery.sol:71:   require(deadline >= block.timestamp, "BaseV1Router: EXPIRED");
```

Recommended Mitigation Steps:

-   Block timestamps should not be used for entropy or generating random numbers—i.e., they should not be the deciding factor (either directly or through some derivation) for winning a game or changing an important state.
-   Time-sensitive logic is sometimes required; e.g., for unlocking contracts (time-locking), completing an ICO after a few weeks, or enforcing expiry dates. It is sometimes recommended to use block.number and an average block time to estimate times; with a 10 second block time, 1 week equates to approximately, 60480 blocks. Thus, specifying a block number at which to change a contract state can be more secure, as miners are unable to easily manipulate the block number.

<br>
<hr>

## 11. Inconsistent method of specifying a floating pragma

_Some files use `>=`, some use `^`. The instances below are examples of the method that has the fewest instances for a specific version. Note that using `>=` without also specifying `<=` will lead to failures to compile, or external project incompatability, when the major version changes and there are breaking-changes, so `^` should be preferred regardless of the instance counts_

```java
  2:    pragma solidity ^0.8.11;
```

<br>
<hr>

## 12. Non-library/interface files should use fixed compiler versions, not floating ones

```java
  2:    pragma solidity ^0.8.11;
```

<br>
<hr>

## 13. Constants in comparisons should appear on the left side

_Constants on the left are better, but this is often trumped by a preference for English word order_

_Doing so will prevent [typo bugs](https://www.moserware.com/2008/01/constants-on-left-are-better-but-this.html)_

Typically we all write comparison statements like this:

```java
if (currentValue == 5)
{
    // do work
}
```

But, the following is just as valid:

```java
if (5 == currentValue)
{
    // do work
}
```

<br>
<hr>

## 14. According to the syntax rules, use `=> mapping ( ` instead of `=> mapping(` using spaces as keyword

```java
-  372: mapping ( address => mapping( uint256 => StakedS1Citizen )) public stakedS1;
+  372: mapping ( address => mapping ( uint256 => StakedS1Citizen )) public stakedS1;


- 405: 	mapping ( address => mapping( uint256 => StakedS2Citizen )) public stakedS2;
+ 405: 	mapping ( address => mapping ( uint256 => StakedS2Citizen )) public stakedS2;

```

<br>
<hr>

## 15. Inconsistent Solidity Versions

_Different Solidity compiler versions are used, the following contracts mix versions_

```java
2: pragma solidity ^0.8.19;
10: pragma solidity ^0.8.0;
10: pragma solidity 0.8.11;
```

Recommendation:

Versions must be consistent with each other.

<br>
<hr>

## 16. Shorthand way to write if / else statement

_The normal `if` / `else` statement can be refactored in a shorthand way to write it:_

1. Increases readability
2. Shortens the overall SLOC

```java
142:  function changeCreateQuestRole(address account_, bool canCreateQuest_) public onlyOwner {
143:        if (canCreateQuest_) {
144:            _grantRole(CREATE_QUEST_ROLE, account_);
145:        } else {
146:            _revokeRole(CREATE_QUEST_ROLE, account_);
147:        }
148:    }
```

The above instance can be refactored in:

```js
function changeCreateQuestRole(address account_, bool canCreateQuest_) public onlyOwner {
        canCreateQuest_ ? _grantRole(CREATE_QUEST_ROLE, account_); : _revokeRole(CREATE_QUEST_ROLE, account_);
    }
```

<br>
<hr>

## 17. Adding a return statement when the function defines a named return variable, is redundant.

```java
function validateSignedSet(RRSetWithSignature memory input, bytes memory proof, uint256 now) internal view returns(RRUtils.SignedSet memory rrset)
{...
return rrset;
    }
```

<br>
<hr>

## 18. Constant redefined elsewhere.

_Consider defining in only one contract so that values cannot become out of sync when only one location is updated. A [cheap way](https://plaxion.medium.com/gas-cost-of-solidity-library-functions-dbe0cedd4678) to store constants in a single location is to create an `internal constant` in a `library`. If the variable is a local cache of another contract's value, consider making the cache variable internal or private, which will require external users to query the contract with the source of truth, so that callers don't get out of sync._

```java
File: src/Pages.sol

/// @audit seen in src/ArtGobblers.sol
86:       Goo public immutable goo;

/// @audit seen in src/ArtGobblers.sol
89:       address public immutable community;

/// @audit seen in src/ArtGobblers.sol
103:      uint256 public immutable mintStart;
```

```java
File: src/utils/rand/ChainlinkV1RandProvider.sol

/// @audit seen in src/utils/token/PagesERC721.sol
20:       ArtGobblers public immutable artGobblers;
```

```java
File: script/deploy/DeployRinkeby.s.sol

/// @audit seen in src/Pages.sol
11:       uint256 public immutable mintStart = 1656369768;
```

## 19. Don’t use `_msgSender()` if not supporting `EIP-2771`.

_Use msg.sender if the code does not implement [EIP-2771 trusted forwarder](https://eips.ethereum.org/EIPS/eip-2771) support._

<br>
<hr>

## 20. The `nonReentrant` modifier should occur before all other modifiers.

_This is a best-practice to protect against reentrancy in other modifiers._

<br>
<hr>

## 21. Function ordering does not follow the Solidity style guide.

_According to the S[olidity style guide](https://docs.soliditylang.org/en/v0.8.17/style-guide.html#order-of-functions), functions should be laid out in the following order: `constructor()`, `receive()`, `fallback()`, `external`, `public`, `internal`, `private`, but the cases below do not follow this pattern:_

```java
File: src/contracts/core/StrategyManager.sol

/// @audit _setStrategyWhitelister() came earlier
857:      function getDeposits(address depositor) external view returns (IStrategy[] memory, uint256[] memory) {
```

[Code example](https://github.com/code-423n4/2023-04-eigenlayer/blob/398cc428541b91948f717482ec973583c9e76232/src/contracts/core/StrategyManager.sol#L857)

<br>
<hr>

## 22. Contract does not follow the Solidity style guide's suggested layout ordering.

_The [style guide](https://docs.soliditylang.org/en/v0.8.16/style-guide.html#order-of-layout) says that, within a contract, the ordering should be 1) `Type declarations`, 2) `State variables`, 3) `Events`, 4) `Modifiers`, and 5) `Functions`, but the contract(s) below do not follow this ordering:_

```java
File: src/contracts/strategies/StrategyBase.sol

/// @audit function _tokenBalance came earlier
250:      uint256[48] private __gap;
```

[Code example](https://github.com/code-423n4/2023-04-eigenlayer/blob/398cc428541b91948f717482ec973583c9e76232/src/contracts/strategies/StrategyBase.sol#L250)

<br>
<hr>

## 23. Avoid the use of sensitive terms.

_Use [alternative variants](https://www.zdnet.com/article/mysql-drops-master-slave-and-blacklist-whitelist-terminology/), e.g. allowlist/denylist instead of whitelist/blacklist_

```java
File: src/contracts/core/StrategyManager.sol

146:      function initialize(address initialOwner, address initialStrategyWhitelister, IPauserRegistry _pauserRegistry, uint256 initialPausedStatus, uint256 _withdrawalDelayBlocks)

846:      function _setStrategyWhitelister(address newStrategyWhitelister) internal {

592:      function addStrategiesToDepositWhitelist(IStrategy[] calldata strategiesToWhitelist) external onlyStrategyWhitelister {

593:          uint256 strategiesToWhitelistLength = strategiesToWhitelist.length;

607:      function removeStrategiesFromDepositWhitelist(IStrategy[] calldata strategiesToRemoveFromWhitelist) external onlyStrategyWhitelister {

608:          uint256 strategiesToRemoveFromWhitelistLength = strategiesToRemoveFromWhitelist.length;

587:      function setStrategyWhitelister(address newStrategyWhitelister) external onlyOwner {

592:      function addStrategiesToDepositWhitelist(IStrategy[] calldata strategiesToWhitelist) external onlyStrategyWhitelister {

607:      function removeStrategiesFromDepositWhitelist(IStrategy[] calldata strategiesToRemoveFromWhitelist) external onlyStrategyWhitelister {

846:      function _setStrategyWhitelister(address newStrategyWhitelister) internal {

84:       /// @notice Emitted when the `strategyWhitelister` is changed

143:       * @param initialStrategyWhitelister The initial value of `strategyWhitelister` to set.

586:      /// @notice Owner-only function to change the `strategyWhitelister` address.

591:      /// @notice Owner-only function that adds the provided Strategies to the 'whitelist' of strategies that stakers can deposit into

595:              // change storage and emit event only if strategy is not already in whitelist

606:      /// @notice Owner-only function that removes the provided Strategies from the 'whitelist' of strategies that stakers can deposit into

610:              // change storage and emit event only if strategy is already in whitelist

845:      /// @notice Internal function for modifying the `strategyWhitelister`. Used inside of the `setStrategyWhitelister` and `initialize` functions.
```

<br>
<hr>

## 24. Typos

Example:

```java
/// @audit usefull
60:           uint256 nonce; // nonce of order usefull for cancelling in bulk
```

<br>
<hr>

## 25. NatSpec is incomplete

```java
File: contracts/core/GolomTrader.sol

/// @audit Missing: '@return'
162       ///      OrderStatus = 3 , valid order
163       /// @param o the Order struct to be validated
164       function validateOrder(Order calldata o)
165           public
166           view
167           returns (
168               uint256,
169               bytes32,
170:              uint256

/// @audit Missing: '@param tokenId'
/// @audit Missing: '@param proof'
328       /// @dev function to fill a signed order of ordertype 2 also has a payment param in case the taker wants
329       ///      to send ether to that address on filling the order, Match an criteria order, ensuring that the supplied proof demonstrates inclusion of the tokenId in the associated merkle root, if root is 0 then any token can be used to fill the order
330       /// @param o the Order struct to be filled must be orderType 2
331       /// @param amount the amount of times the order is to be filled(useful for ERC1155)
332       /// @param referrer referrer of the order
333       /// @param p any extra payment that the taker of this order wanna send on succesful execution of order
334       function fillCriteriaBid(
335           Order calldata o,
336           uint256 amount,
337           uint256 tokenId,
338           bytes32[] calldata proof,
339           address referrer,
340           Payment calldata p
341:      ) public nonReentrant {
```

## 26.1 Missing NatSpec in file.

<br>
<hr>

## 27. Inconsistent spacing in comments

_Some lines use `// x` and some use `//x`. The instances below point out the usages that don’t follow the majority, within each file_

Example:

```java
File: contracts/core/GolomTrader.sol
181:          //deadline

File: contracts/rewards/RewardDistributor.sol
99:           //console.log(block.timestamp,epoch,fee);
```

<br>
<hr>

## 28. Consider disabling `renounceOwnership()`.

_If the plan for your project does not include eventually giving up all ownership control, consider overwriting OpenZeppelin's `Ownable`'s `renounceOwnership()` function in order to disable it._

[Example](https://github.com/code-423n4/2023-05-venus/blob/9853f6f4fe906b635e214b22de9f627c6a17ba5b/contracts/Comptroller.sol#L18-L18):

```java
File: contracts/Comptroller.sol

18:      Ownable2StepUpgradeable,
```

[Example](https://github.com/code-423n4/2023-05-venus/blob/9853f6f4fe906b635e214b22de9f627c6a17ba5b/contracts/Pool/PoolRegistry.sol#L25-L25):

```java
File: contracts/Pool/PoolRegistry.sol

25:  contract PoolRegistry is Ownable2StepUpgradeable, AccessControlledV8, PoolRegistryInterface {
```

<br>
<hr>
<br>

based on real reports [Code4arena](https://code4rena.com/reports)
