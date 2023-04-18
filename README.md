# QA-report-Low-and-Non-Critical

## Non-Critical

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
<br>

based on real reports [Code4arena](https://code4rena.com/reports)
