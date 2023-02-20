## [G-01] INCREMENTS CAN BE UNCHECKED

In Solidity 0.8+, there’s a default overflow check on unsigned integers. It’s possible to uncheck this in for-loops and save some gas at each iteration, but at the cost of some code readability, as this uncheck cannot be made inline.

https://github.com/ethereum/solidity/issues/10695

Instances include:

```
MaltRepository.grantRoleMultiple()#132:    for (uint256 i; i < length; ++i) {
MaltRepository._setup()#188:    for (uint256 i; i < length; ++i) {
RewardThrottle.checkRewardUnderflow()#446:      for (uint256 i = _activeEpoch; i < epoch; ++i) {
RewardThrottle._sendToDistributor()#590:    for (uint256 i; i < length; ++i) {
RewardThrottle._fillInEpochGaps()#639:    for (uint256 i = _activeEpoch + 1; i <= epoch; ++i) {
RewardThrottle.populateFromPreviousThrottle()#667:    for (uint256 i = _activeEpoch; i < epoch; ++i) {
SwingTraderManager.buyMalt()#154:    for (uint256 i; i < length; ++i) {
SwingTraderManager.buyMalt()#170:    for (uint256 i; i < length; ++i) {
SwingTraderManager.sellMalt()#208:    for (uint256 i; i < length; ++i) {
SwingTraderManager.sellMalt()#224:    for (uint256 i; i < length; ++i) {
SwingTraderManager.costBasis()#269:    for (uint256 i; i < length; ++i) {
SwingTraderManager.calculateSwingTraderMaltRatio()#300:    for (uint256 i; i < length; ++i) {
SwingTraderManager.getTokenBalances()#330:    for (uint256 i; i < length; ++i) {
SwingTraderManager.delegateCapital()#348:    for (uint256 i; i < length; ++i) {
SwingTraderManager.delegateCapital()#366:    for (uint256 i; i < length; ++i) {
SwingTraderManager.deployedCapital()#389:    for (uint256 i; i < length; ++i) {
```

The code would go from:

```
for (uint256 i; i < numIterations; ++i) {  
 // ...  
}  
```

to

```
for (uint256 i; i < numIterations;) {  
 // ...  
 unchecked { ++i; }  
}  
```


## [G-02] GlobalImpliedCollateralService.swingTraderCollateralRatio(): SHOULD USE MEMORY INSTEAD OF STORAGE VARIABLE

See @audit tag

```solidity
  function swingTraderCollateralRatio() public view returns (uint256) {
    uint256 decimals = malt.decimals();
    uint256 totalSupply = malt.totalSupply();

    if (totalSupply == 0) {
      return 0;
    }

    return (collateral.swingTrader * (10**decimals)) / malt.totalSupply(); // @audit gas: should use  '... / totalSupply'
  }
```



## [G-03] SwingTraderManager.buyMalt(): SHOULD USE MEMORY INSTEAD OF STORAGE VARIABLE

See @audit tag

```solidity
    uint256[] memory traderIds = activeTraders;
    uint256 length = traderIds.length;

    uint256 totalCapital;
    uint256[] memory traderCapital = new uint256[](length);

    for (uint256 i; i < length; ++i) {
      SwingTraderData memory trader = swingTraders[activeTraders[i]];  // @audit gas: should use  'swingTraders[traderIds[i]]'

      if (!trader.active) {
        continue;
      }

      uint256 traderBalance = collateralToken.balanceOf(trader.traderContract);
      totalCapital += traderBalance;
      traderCapital[i] = traderBalance;
    }

    if (totalCapital == 0) {
      return 0;
    }

    for (uint256 i; i < length; ++i) {
      SwingTraderData memory trader = swingTraders[activeTraders[i]]; // @audit gas: should use  'swingTraders[traderIds[i]]'
      uint256 share = (maxCapital * traderCapital[i]) / totalCapital;

      if (share == 0) {
        continue;
      }
```


## [G-04] SwingTraderManager.sellMalt(): SHOULD USE MEMORY INSTEAD OF STORAGE VARIABLE

See @audit tag

```solidity
    uint256[] memory traderIds = activeTraders;
    uint256 length = traderIds.length;
    uint256 profit;

    uint256 totalMalt;
    uint256[] memory traderMalt = new uint256[](length);

    for (uint256 i; i < length; ++i) {
      SwingTraderData memory trader = swingTraders[activeTraders[i]];   // @audit gas: should use  'swingTraders[traderIds[i]]'

      if (!trader.active) {
        continue;
      }

      uint256 traderMaltBalance = malt.balanceOf(trader.traderContract);
      totalMalt += traderMaltBalance;
      traderMalt[i] = traderMaltBalance;
    }

    if (totalMalt == 0) {
      return 0;
    }

    for (uint256 i; i < length; ++i) {
      SwingTraderData memory trader = swingTraders[activeTraders[i]];   // @audit gas: should use  'swingTraders[traderIds[i]]'
```

## [G-05] SwingTraderManager.costBasis(): SHOULD USE MEMORY INSTEAD OF STORAGE VARIABLE

See @audit tag

```solidity
    uint256[] memory traderIds = activeTraders;
    uint256 length = traderIds.length;
    decimals = collateralToken.decimals();

    uint256 totalMaltBalance;
    uint256 totalDeployedCapital;

    for (uint256 i; i < length; ++i) {
      SwingTraderData memory trader = swingTraders[activeTraders[i]]; // @audit gas: should use  'swingTraders[traderIds[i]]'
```



## [G-06] SwingTraderManager.calculateSwingTraderMaltRatio(): SHOULD USE MEMORY INSTEAD OF STORAGE VARIABLE

See @audit tag

```solidity
  function calculateSwingTraderMaltRatio()
    public
    view
    returns (uint256 maltRatio)
  {
    uint256[] memory traderIds = activeTraders;
    uint256 length = traderIds.length;
    uint256 decimals = collateralToken.decimals();
    uint256 maltDecimals = malt.decimals();
    uint256 totalMaltBalance;
    uint256 totalCollateralBalance;

    for (uint256 i; i < length; ++i) {
      SwingTraderData memory trader = swingTraders[activeTraders[i]]; // @audit gas: should use  'swingTraders[traderIds[i]]'
```


## [G-07] SwingTraderManager.getTokenBalances(): SHOULD USE MEMORY INSTEAD OF STORAGE VARIABLE

See @audit tag

```solidity
  function getTokenBalances()
    external
    view
    returns (uint256 maltBalance, uint256 collateralBalance)
  {
    uint256[] memory traderIds = activeTraders;
    uint256 length = traderIds.length;

    for (uint256 i; i < length; ++i) {
      SwingTraderData memory trader = swingTraders[activeTraders[i]];  // @audit gas: should use  'swingTraders[traderIds[i]]'
```


## [G-08] SwingTraderManager.delegateCapital(): SHOULD USE MEMORY INSTEAD OF STORAGE VARIABLE

See @audit tag

```solidity
  function delegateCapital(uint256 amount, address destination)
    external
    onlyRoleMalt(CAPITAL_DELEGATE_ROLE, "Must have capital delegation privs")
    onlyActive
  {
    uint256[] memory traderIds = activeTraders;
    uint256 length = traderIds.length;

    uint256 totalCapital;
    uint256[] memory traderCapital = new uint256[](length);

    for (uint256 i; i < length; ++i) {
      SwingTraderData memory trader = swingTraders[activeTraders[i]];   // @audit gas: should use  'swingTraders[traderIds[i]]'

      if (!trader.active) {
        continue;
      }

      uint256 traderBalance = collateralToken.balanceOf(trader.traderContract);
      totalCapital += traderBalance;
      traderCapital[i] = traderBalance;
    }

    if (totalCapital == 0) {
      return;
    }

    uint256 capitalUsed;

    for (uint256 i; i < length; ++i) {
      SwingTraderData memory trader = swingTraders[activeTraders[i]];    // @audit gas: should use  'swingTraders[traderIds[i]]'
```



## [G-09] SwingTraderManager.deployedCapital(): SHOULD USE MEMORY INSTEAD OF STORAGE VARIABLE

See @audit tag

```
  function deployedCapital() external view returns (uint256 deployed) {
    uint256[] memory traderIds = activeTraders;
    uint256 length = traderIds.length;

    for (uint256 i; i < length; ++i) {
      SwingTraderData memory trader = swingTraders[activeTraders[i]];    // @audit gas: should use  'swingTraders[traderIds[i]]'
```



## [G-10] GlobalImpliedCollateralService.sync(): existingPool.* SHOULD GET CACHED

See @audit tag

```solidity
    uint256 existingCollateral = existingPool.total;

    uint256 total = collateral.total; // gas
    if (existingCollateral <= total) {
      total -= existingCollateral; // subtract existing value
    } else {
      total = 0;
    }

    uint256 swingTraderMalt = collateral.swingTraderMalt; // gas  
    if (existingPool.swingTraderMalt <= swingTraderMalt) {        //@audit gas: should cache "existingPool.swingTraderMalt" (SLOAD 1)
      swingTraderMalt -= existingPool.swingTraderMalt;            //@audit gas: should cache "existingPool.swingTraderMalt" (SLOAD 2)
    } else {
      swingTraderMalt = 0;
    }

    uint256 swingTraderCollat = collateral.swingTrader; // gas
    if (existingPool.swingTrader <= swingTraderCollat) {         //@audit gas: should cache "existingPool.swingTrader" (SLOAD 1)
      swingTraderCollat -= existingPool.swingTrader;             //@audit gas: should cache "existingPool.swingTrader" (SLOAD 2)
    } else {
      swingTraderCollat = 0;
    }

    uint256 arb = collateral.arbTokens; // gas
    if (existingPool.arbTokens <= arb) {                         //@audit gas: should cache "existingPool.arbTokens" (SLOAD 1)
      arb -= existingPool.arbTokens;                             //@audit gas: should cache "existingPool.arbTokens" (SLOAD 2)
    } else {
      arb = 0;
    }

    uint256 overflow = collateral.rewardOverflow; // gas
    if (existingPool.rewardOverflow <= overflow) {               //@audit gas: should cache "existingPool.rewardOverflow" (SLOAD 1)
      overflow -= existingPool.rewardOverflow;                   //@audit gas: should cache "existingPool.rewardOverflow" (SLOAD 2)
    } else {
      overflow = 0;
    }

    uint256 liquidityExtension = collateral.liquidityExtension; // gas
    if (existingPool.liquidityExtension <= liquidityExtension) { //@audit gas: should cache "existingPool.liquidityExtension" (SLOAD 1)
      liquidityExtension -= existingPool.liquidityExtension;     //@audit gas: should cache "existingPool.liquidityExtension" (SLOAD 2)
    } else {
      liquidityExtension = 0;
    }
```

## [G-11] LinearDistributor.decrementRewards(): declaredBalance SHOULD GET CACHED

See @audit tag

```solidity
  function decrementRewards(uint256 amount)
    external
    onlyRoleMalt(REWARD_MINE_ROLE, "Only reward mine")
  {
    require(
      amount <= declaredBalance,                         //@audit gas: should cache "declaredBalance" (SLOAD 1)
      "Can't decrement more than total reward balance"
    );

    if (amount > 0) {
      declaredBalance = declaredBalance - amount;        //@audit gas: should cache "declaredBalance" (SLOAD 2)
    }
  }
```

## [G-12] LinearDistributor._forfeit(): declaredBalance SHOULD GET CACHED

See @audit tag

```solidity
  function _forfeit(uint256 forfeited) internal {
    require(forfeited <= declaredBalance, "Cannot forfeit more than declared");  //@audit gas: should cache "declaredBalance" (SLOAD 1)

    declaredBalance = declaredBalance - forfeited;                               //@audit gas: should cache "declaredBalance" (SLOAD 2)
```

## [G-13] SwingTraderManager.buyMalt(): swingTraders SHOULD GET CACHED

See @audit tag

```solidity
    for (uint256 i; i < length; ++i) {
      SwingTraderData memory trader = swingTraders[activeTraders[i]];          //@audit gas: should cache "swingTraders" (SLOAD 1)

      if (!trader.active) {
        continue;
      }

      uint256 traderBalance = collateralToken.balanceOf(trader.traderContract);
      totalCapital += traderBalance;
      traderCapital[i] = traderBalance;
    }

    if (totalCapital == 0) {
      return 0;
    }

    for (uint256 i; i < length; ++i) {
      SwingTraderData memory trader = swingTraders[activeTraders[i]];           //@audit gas: should cache "swingTraders" (SLOAD 2)
```

## [G-14] SwingTraderManager.sellMalt(): swingTraders SHOULD GET CACHED

See @audit tag

```solidity
      SwingTraderData memory trader = swingTraders[activeTraders[i]];           //@audit gas: should cache "swingTraders" (SLOAD 1)

      if (!trader.active) {
        continue;
      }

      uint256 traderMaltBalance = malt.balanceOf(trader.traderContract);
      totalMalt += traderMaltBalance;
      traderMalt[i] = traderMaltBalance;
    }

    if (totalMalt == 0) {
      return 0;
    }

    for (uint256 i; i < length; ++i) {
      SwingTraderData memory trader = swingTraders[activeTraders[i]];          //@audit gas: should cache "swingTraders" (SLOAD 2)
      uint256 share = (maxAmount * traderMalt[i]) / totalMalt;
```


## [G-15] SwingTraderManager.delegateCapital(): swingTraders SHOULD GET CACHED

See @audit tag

```solidity
    for (uint256 i; i < length; ++i) {
      SwingTraderData memory trader = swingTraders[activeTraders[i]];         //@audit gas: should cache "swingTraders" (SLOAD 1)

      if (!trader.active) {
        continue;
      }

      uint256 traderBalance = collateralToken.balanceOf(trader.traderContract);
      totalCapital += traderBalance;
      traderCapital[i] = traderBalance;
    }

    if (totalCapital == 0) {
      return;
    }

    uint256 capitalUsed;

    for (uint256 i; i < length; ++i) {
      SwingTraderData memory trader = swingTraders[activeTraders[i]];         //@audit gas: should cache "swingTraders" (SLOAD 2)
      uint256 share = (amount * traderCapital[i]) / totalCapital;
```


## [G-16] LinearDistributor.sol has code that needs to be UNCHECKED in many places
Solidity version 0.8+ comes with implicit overflow and underflow checks on unsigned integers. When an overflow or an underflow isn’t possible (as an example, when a comparison is made before the arithmetic operation), some gas can be saved by using an unchecked block: https://docs.soliditylang.org/en/v0.8.7/control-structures.html#checked-or-unchecked-arithmetic
L149 SHOULD BE UNCHECKED DUE TO L147
L169 SHOULD BE UNCHECKED DUE TO L164
L188 SHOULD BE UNCHECKED DUE TO L186
```solidity
147:    if (balance > bufferRequirement) {
148:      // We have more than the buffer required. Forfeit the rest
149:      uint256 net = balance - bufferRequirement;
150:      _forfeit(net);
151:    }
...
163:    require(
164:      amount <= declaredBalance,
165:      "Can't decrement more than total reward balance"
166:    );
167:
168:    if (amount > 0) {
169:      declaredBalance = declaredBalance - amount;
170:    }
...
185:  function _forfeit(uint256 forfeited) internal {
186:    require(forfeited <= declaredBalance, "Cannot forfeit more than declared");
187:
188:    declaredBalance = declaredBalance - forfeited;
```

## [G-17] RewardThrottle.sol has code that needs to be UNCHECKED in many places
L117 SHOULD BE UNCHECKED DUE TO L116
L162 AND L146 SHOULD BE UNCHECKED DUE TO L145
L222 SHOULD BE UNCHECKED DUE TO L219
...

```
116:        if (balance > remainder) {
117          balance -= remainder;
...
145    if (cashflowAverageApr > targetCashflowApr) {
146      uint256 delta = cashflowAverageApr - targetCashflowApr;
...
162:      uint256 delta = targetCashflowApr - cashflowAverageApr;
...
219    if (endEpoch < averagePeriod) {
      averagePeriod = currentEpoch;
    } else {
222:      startEpoch = endEpoch - averagePeriod;
...
```