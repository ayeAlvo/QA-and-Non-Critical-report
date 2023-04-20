# QA report: Low and Non Critical

# Non-Critical

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

## 9. Consider addings checks for signature malleability

_Use OpenZeppelin's `ECDSA` contract rather than calling `ecrecover()` directly_

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
<br>

# Low Risk

## 1. Don’t use `payable.transfer()`/`payable.send()`

_The use of `payable.transfer()` is [heavily frowned upon](https://consensys.net/diligence/blog/2019/09/stop-using-soliditys-transfer-now/) because it can lead to the locking of funds. The `transfer()` call requires that the recipient is either an EOA account, or is a contract that has a `payable` callback. For the contract case, the `transfer()` call only provides 2300 gas for the contract to complete its operations. This means the following cases can cause the transfer to fail:_

-   The contract does not have a `payable` callback.
-   The contract’s `payable` callback spends more than 2300 gas (which is only enough to emit something).
-   The contract is called through a proxy which itself uses up the 2300 gas Use OpenZeppelin’s `Address.sendValue()` instead.
-   Future changes or forks in Ethereum result in higher gas fees than transfer provides. The .transfer() creates a hard dependency on 2300 gas units being appropriate now and into the future.

Example wrong:

```java
payable(payAddress).transfer(payAmt);
```

Instead use the `.call()` function to transfer ether and avoid some of the limitations of .`transfer()`;

Example:

```java
(bool success, ) = payable(payAddress).call{value: payAmt}(""); // royalty transfer to royaltyaddress
require(success, "Transfer failed.");
```

Note: Gas units can also be passed to the `.call()` function as a variable to accomodate any uses edge cases. Gas could be a mutable state variable that can be set by the contract owner.

<br>
<hr>

## 2. Unused/empty `receive()`/`fallback()` function

_If the intention is for the Ether to be used, the function should call another function, otherwise it should revert (e.g. `require(msg.sender == address(weth))`)_

Example:

```java
fallback() external payable {}
receive() external payable {}
```

<br>
<hr>

## 3. Use Custom Errors instead of String

_Use custom errors to save deployment and runtime costs in case of revert._

_Instead of using strings for error messages (e.g., `require(msg.sender == owner, “unauthorized”)`), you can use custom errors to reduce both deployment and runtime gas costs. In addition, they are very convenient as you can easily pass dynamic information to them._

Before:

```java
function add(uint256 _amount) public {
    require(msg.sender == owner, "unauthorized");

    total += _amount;
}
```

```java
 require(shares + ONE_DEC18 < totalShares, "too many shares");
```

After:

```java
error Unauthorized(address caller);

function add(uint256 _amount) public {
    if (msg.sender != owner)
        revert Unauthorized(msg.sender);

    total += _amount;
}
```

<br>
<hr>

## 4. `require()` should be used instead of `assert()`

_Prior to solidity version 0.8.0, hitting an assert consumes the remainder of the transaction’s available gas rather than returning it, as `require()`/`revert()`._

_`assert()` should be avoided even past solidity version 0.8.0 as its [documentation](https://docs.soliditylang.org/en/v0.8.14/control-structures.html#panic-via-assert-and-error-via-require) states that “The assert function creates an error of type Panic(uint256). … Properly functioning code should never create a Panic, not even on invalid external input. If this happens, then there is a bug in your contract which you should fix”._

Example:

```java
493:          assert(idToOwner[_tokenId] == address(0));
506:          assert(idToOwner[_tokenId] == _from);
```

<br>
<hr>

## 5. Unsafe use of `transfer()`/`transferFrom()` with `IERC20`

_Some tokens do not implement the ERC20 standard properly but are still accepted by most code that accepts ERC20 tokens. For example Tether (USDT)'s `transfer()` and `transferFrom()` functions do not return booleans as the specification requires, and instead have no return value. When these sorts of tokens are cast to `IERC20`, their function signatures do not match and therefore the calls made, revert. Use OpenZeppelin’s SafeERC20's `safeTransfer()`/`safeTransferFrom()` instead_

[Example](https://github.com/code-423n4/2022-05-rubicon/blob/8c312a63a91193c6a192a9aab44ff980fbfd7741/contracts/rubiconPools/BathPair.sol#L601):

```java
File: contracts/rubiconPools/BathPair.sol

601:              IERC20(asset).transfer(msg.sender, booty);

615:              IERC20(quote).transfer(msg.sender, booty);
```

<br>
<hr>

## 6. Return values of `transfer()`/`transferFrom()` not checked

_Not all `IERC20` implementations `revert()` when there's a failure in `transfer()`/`transferFrom()`. The function signature has a `boolean` return value and they indicate errors that way instead. By not checking the return value, operations that should have marked as failed, may potentially go through without actually making a payment_

[Example](https://github.com/code-423n4/2022-05-rubicon/blob/8c312a63a91193c6a192a9aab44ff980fbfd7741/contracts/rubiconPools/BathPair.sol#L601): (same code as before)

```java
File: contracts/rubiconPools/BathPair.sol

601:              IERC20(asset).transfer(msg.sender, booty);

615:              IERC20(quote).transfer(msg.sender, booty);

```

<br>
<hr>

## 7. Return values of `approve()` not checked

_Not all `IERC20` implementations `revert()` when there's a failure in `approve()`. The function signature has a `boolean` return value and they indicate errors that way instead. By not checking the return value, operations that should have marked as failed, may potentially go through without actually approving anything_

[Example:](https://github.com/code-423n4/2022-05-rubicon/blob/8c312a63a91193c6a192a9aab44ff980fbfd7741/contracts/rubiconPools/BathToken.sol#L214)

```java
File: contracts/rubiconPools/BathToken.sol   #1

214:   IERC20(address(token)).approve(RubiconMarketAddress, 2**256 - 1);
```

Recommended Mitigation Steps:

-   It is recommend to use `safeApprove()`.
-   Consider using `safeIncreaseAllowance` instead of approve function. (Approve race condition)

<br>
<hr>

## 8. Typos

Example:

```java
/// @audit usefull
60:           uint256 nonce; // nonce of order usefull for cancelling in bulk
```

<br>
<hr>

## 9. NatSpec is incomplete

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

<br>
<hr>

## 10. Inconsistent spacing in comments

_Some lines use `// x` and some use `//x`. The instances below point out the usages that don’t follow the majority, within each file_

Example:

```java
File: contracts/core/GolomTrader.sol
181:          //deadline

File: contracts/rewards/RewardDistributor.sol
99:           //console.log(block.timestamp,epoch,fee);
```

## 11. Missing checks for `address(0x0)` (zero-address) when assigning values to `address` state variables

_Missing checks for zero-addresses may lead to infunctional protocol, if the variable addresses are updated incorrectly._
Example:

`factory = _factory;`

```java
constructor(
        address _factory,
        INonfungiblePositionManager _nonfungiblePositionManager,
        ISwapRouter _swapRouter,
        ComptrollerInterface _comptroller
    ) {
        factory = _factory;
        nonfungiblePositionManager = _nonfungiblePositionManager;
        swapRouter = _swapRouter;
        comptroller = _comptroller;
        _notEntered = true;
    }
```

<br>

`pendingDistributor = _distributor;`

```java
/// @notice Sets the distributor contract
    /// @param _distributor Address of the distributor
    function setDistributor(address _distributor) external onlyOwner {
        if (address(distributor) == address(0)) {
            distributor = Distributor(_distributor);
        } else {
            pendingDistributor = _distributor;
            distributorEnableDate = block.timestamp + 1 days;
        }
    }
```

Recommended Mitigation Steps:

-   Consider adding zero-address checks:
    `require(newAddr != address(0));`

<br>
<hr>

## 12. The Contract Should `approve(0)`/`safeApprove()` first - A second call to the function may revert

_Some ERC20 tokens, such as Tether, `revert()` if `approve()` is called a second time without having called `approve(0)` first_

_Some tokens (like USDT L199) do not work when changing the allowance from an existing non-zero allowance value.
They must first be approved by zero and then the actual allowance must be approved._

_When trying to re-approve an already approved token, all transactions revert and the protocol cannot be used._

Recommended Mitigation Steps:

-   Approve with a zero amount first before setting the actual amount. Consider use `safeIncreaseAllowance` and `safeDecreaseAllowance`.

Example:

```java
IERC20(token).safeApprove(address(operator), 0);
IERC20(token).safeApprove(address(operator), amount);
```

<br>
<hr>
<br>

based on real reports [Code4arena](https://code4rena.com/reports)
