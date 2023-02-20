### [G-01] Unnecessary calculation
- https://github.com/code-423n4/2023-02-malt/blob/main/contracts/RewardSystem/LinearDistributor.sol#L142

```solidity
bufferRequirement = (distributed * buf * 10000) / netTime / 10000;
```

We can omit `* 10000` and `/ 10000` as the result will be same.

### [G-02] Needless declaration
- https://github.com/code-423n4/2023-02-malt/blob/main/contracts/StabilityPod/StabilizerNode.sol#L291

```solidity
uint256 decimals = collateralToken.decimals();
```

`decimals` are declared but isn't used at all.

### [G-03] Checking `if` conditoins that can't meet
- https://github.com/code-423n4/2023-02-malt/blob/main/contracts/StabilityPod/SwingTraderManager.sol#L178

```solidity
      if (capitalUsed + share > maxCapital) { //@audit-issue this condition won't meet
        share = maxCapital - capitalUsed;
      }

      uint256 used = ISwingTrader(trader.traderContract).buyMalt(share);
      capitalUsed += used;
```

From the `buyMalt()` logic, `used <= share` and `capitalUsed + share` can't be greater than `maxCapital` for any cases.

- https://github.com/code-423n4/2023-02-malt/blob/main/contracts/StabilityPod/SwingTraderManager.sol#L232

```solidity
      if (amountSold + share > maxAmount) { //@audit-issue needless check
        share = maxAmount - amountSold;
      }

      uint256 initialProfit = ISwingTrader(trader.traderContract).totalProfit();
      try ISwingTrader(trader.traderContract).sellMalt(share) returns (
        uint256 sold
      ) {
        uint256 finalProfit = ISwingTrader(trader.traderContract).totalProfit();
        profit += (finalProfit - initialProfit);
        amountSold += sold;
```

From the `sellMalt()` logic, `sold <= share` and `amountSold + share` can't be greater than `maxAmount`.


### [G-04] `delegateCapital()` should check the balances to save gas.
- https://github.com/code-423n4/2023-02-malt/blob/main/contracts/StabilityPod/SwingTraderManager.sol#L379

```solidity
ISwingTrader(trader.traderContract).delegateCapital(share, destination);
```

`SwingTrader.delegateCapital()` tries to transfer without checking contract balance and it will revert if the `share` is more than the balance.

So we can revert `SwingTraderManager.delegateCapital()` if `totalCapital < amount` at L363 because it will revert anyway in `SwingTrader.delegateCapital()` after some share calculations.

### [G-05] `x += y` costs more gas than `x = x + y` for state variables
- https://github.com/code-423n4/2023-02-malt/blob/main/contracts/GlobalImpliedCollateralService.sol#L172-L177

```solidity
    total += _pool.total;
    swingTraderMalt += _pool.swingTraderMalt;
    swingTraderCollat += _pool.swingTrader;
    arb += _pool.arbTokens;
    overflow += _pool.rewardOverflow;
    liquidityExtension += _pool.liquidityExtension;
```

- https://github.com/code-423n4/2023-02-malt/blob/main/contracts/RewardSystem/RewardThrottle.sol#L103

```solidity
    state[_activeEpoch].profit += balance;
```

- https://github.com/code-423n4/2023-02-malt/blob/main/contracts/RewardSystem/RewardThrottle.sol#L153

```solidity
    newAPR += adjustment;
```

- https://github.com/code-423n4/2023-02-malt/blob/main/contracts/RewardSystem/RewardThrottle.sol#L604

```solidity
    rewarded += share;
```

### [G-06] Constructors can be marked payable
Payable functions cost less gas to execute, since the compiler does not have to add extra checks to ensure that a payment wasn't provided. A constructor can safely be marked as payable, since only the deployer would be able to pass funds, and the project itself would not pass any funds.

### [G-07] Using calldata instead of memory
- https://github.com/code-423n4/2023-02-malt/blob/main/contracts/GlobalImpliedCollateralService.sol#L120

```solidity
    function sync(PoolCollateral memory _pool) external {
```

- https://github.com/code-423n4/2023-02-malt/blob/main/contracts/Repository.sol#L104

```solidity
    function addNewContract(string memory _name, address _contract)
```

- https://github.com/code-423n4/2023-02-malt/blob/main/contracts/Repository.sol#L111

```solidity
function removeContract(string memory _name)
```

- https://github.com/code-423n4/2023-02-malt/blob/main/contracts/Repository.sol#L118

```solidity
    function updateContract(string memory _name, address _contract)
```
