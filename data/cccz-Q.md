## [Low-01] SwingTraderManager.totalProfit may be updated incorrectly

### Impact
In SwingTraderManager.sellMalt, when amountSold + dustThreshold >= maxAmount, the function will return directly and will not update totalProfit, which will cause totalProfit to update incorrectly
```
    if (amountSold + dustThreshold >= maxAmount) {
      return maxAmount;
    }

    totalProfit += profit;
```
### Proof of Concept

https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/SwingTraderManager.sol#L251-L259

### Recommended Mitigation Steps
Change to
```diff
      if (amountSold >= maxAmount) {
        break;
      }
    }

+   totalProfit += profit;

    if (amountSold + dustThreshold >= maxAmount) {
      return maxAmount;
    }

-   totalProfit += profit;
```

## [Low-02] GlobalImpliedCollateralService.setPoolUpdater does not update poolUpdatersLookup[oldPool]

### Impact

Consider the following situation
poolUpdaters[A] = a
poolUpdatersLookup[a] = A
poolUpdaters[B] = b
poolUpdatersLookup[b] = B

Call GlobalImpliedCollateralService.setPoolUpdater(a, B)
```solidity
  function setPoolUpdater(address _pool, address _updater)
    external
    onlyRoleMalt(UPDATER_MANAGER_ROLE, "Must have updater manager role")
  {
    require(_updater != address(0), "GlobImpCol: No addr(0)");
    poolUpdaters[_updater] = _pool;
    address oldUpdater = poolUpdatersLookup[_pool];
    emit SetPoolUpdater(_pool, _updater);
    poolUpdaters[oldUpdater] = address(0);
    poolUpdatersLookup[_pool] = _updater;
  }
```

poolUpdaters[B] = a
poolUpdaters[A] = 0
poolUpdatersLookup[a] = B

but poolUpdatersLookup[b] = B
Should make poolUpdatersLookup[b] = 0
### Proof of Concept
https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/GlobalImpliedCollateralService.sol#L108-L118

### Recommended Mitigation Steps
Change to
```diff
  function setPoolUpdater(address _pool, address _updater)
    external
    onlyRoleMalt(UPDATER_MANAGER_ROLE, "Must have updater manager role")
  {
    require(_updater != address(0), "GlobImpCol: No addr(0)");
+   address oldPool = poolUpdaters[_updater];
    poolUpdaters[_updater] = _pool;
    address oldUpdater = poolUpdatersLookup[_pool];
    emit SetPoolUpdater(_pool, _updater);
    poolUpdaters[oldUpdater] = address(0);
    poolUpdatersLookup[_pool] = _updater;
+   poolUpdatersLookup[oldPool] = address(0);
  }
```

## [Low-03] Incorrect reason for Permissions.emergencyWithdrawGAS/partialWithdraw

### Impact
The reason for Permissions.emergencyWithdrawGAS/partialWithdraw should be "Must have timelock role"
```solidity
  function emergencyWithdrawGAS(address payable destination)
    external
    onlyRoleMalt(TIMELOCK_ROLE, "Only timelock can assign roles")
  {
    require(destination != address(0), "Withdraw: addr(0)");
    // Transfers the entire balance of the Gas token to destination
    (bool success, ) = destination.call{value: address(this).balance}("");
    require(success, "emergencyWithdrawGAS error");
  }
  ...
  function partialWithdraw(
    address _token,
    address destination,
    uint256 amount
  ) external onlyRoleMalt(TIMELOCK_ROLE, "Only timelock can assign roles") {
    require(destination != address(0), "Withdraw: addr(0)");
    ERC20 token = ERC20(_token);
    token.safeTransfer(destination, amount);
  }
```

### Proof of Concept
https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/Permissions.sol#L46-L50
https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/Permissions.sol#L76-L80
### Recommended Mitigation Steps
Change to
```diff
  function emergencyWithdrawGAS(address payable destination)
    external
-   onlyRoleMalt(TIMELOCK_ROLE, "Only timelock can assign roles")
+   onlyRoleMalt(TIMELOCK_ROLE, "Must have timelock role")

  {
    require(destination != address(0), "Withdraw: addr(0)");
    // Transfers the entire balance of the Gas token to destination
    (bool success, ) = destination.call{value: address(this).balance}("");
    require(success, "emergencyWithdrawGAS error");
  }
  ...
  function partialWithdraw(
    address _token,
    address destination,
    uint256 amount
- ) external onlyRoleMalt(TIMELOCK_ROLE, "Only timelock can assign roles") {
+ ) external onlyRoleMalt(TIMELOCK_ROLE, "Must have timelock role") {

    require(destination != address(0), "Withdraw: addr(0)");
    ERC20 token = ERC20(_token);
    token.safeTransfer(destination, amount);
  }
```