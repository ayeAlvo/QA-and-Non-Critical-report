# QA report: Low and Non Critical

# Non-Critical

## 1. Event is missing `indexed` fields.

_Index event fields make the field more quickly accessible to off-chain tools that parse events. However, note that each index field costs extra gas during emission, so it’s not necessarily best to index the maximum allowed per event (threefields). Each event should use three indexed fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. `If there are fewer than three fields, all of the fields should be indexed`._

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
<br>

# Low Risk

## 1. Don’t use `payable.transfer()`/`payable.send()`

_The use of `payable.transfer()` is [heavily frowned upon](https://consensys.net/diligence/blog/2019/09/stop-using-soliditys-transfer-now/) because it can lead to the locking of funds. The `transfer()` call requires that the recipient is either an EOA account, or is a contract that has a `payable` callback. For the contract case, the `transfer()` call only provides 2300 gas for the contract to complete its operations. This means the following cases can cause the transfer to fail:_

-   The contract does not have a `payable` callback
-   The contract’s `payable` callback spends more than 2300 gas (which is only enough to emit something)
-   The contract is called through a proxy which itself uses up the 2300 gas Use OpenZeppelin’s `Address.sendValue()` instead

Example:

```java
payable(payAddress).transfer(payAmt);
```

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

## 7. Return values of approve() not checked

_Not all `IERC20` implementations `revert()` when there's a failure in `approve()`. The function signature has a `boolean` return value and they indicate errors that way instead. By not checking the return value, operations that should have marked as failed, may potentially go through without actually approving anything_

[Example:](https://github.com/code-423n4/2022-05-rubicon/blob/8c312a63a91193c6a192a9aab44ff980fbfd7741/contracts/rubiconPools/BathToken.sol#L214)

```java
File: contracts/rubiconPools/BathToken.sol   #1

214:   IERC20(address(token)).approve(RubiconMarketAddress, 2**256 - 1);
```

<br>
<hr>
<br>

based on real reports [Code4arena](https://code4rena.com/reports)
