### [L-01] `runwayDays` might be longer than it should be due to possible rounding issue

- https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/RewardSystem/RewardThrottle.sol#L399
- https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/RewardSystem/RewardThrottle.sol#L406

```solidity
        uint256 epochsPerDay = 86400 / timekeeper.epochLength();
        ...
        runwayDays = runwayEpochs / epochsPerDay;
```

When 86400 is not a multiple of `timekeeper.epochLength()`, `runwayDays` might be longer than it should be.
Let us assume that `timekeeper.epochLength()` = 43201 (about half a day), and `runwayEpochs` = 360 (about 180 days).
`runwayDays` should be `runwayEpochs * timekeeper.epochLength() / 86400` = 180, but in the above implementation, `epochsPerDay` = 1 and `runwayDays` = 360. 
It is recommended to use `runwayDays = runwayEpochs * timekeeper.epochLength() / 86400` directly without the middle variable `epochsPerDay`.


### [L-02] `primedBlock` is reset to 0 instead of block.number

- https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/StabilizerNode.sol#L224
- https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/StabilizerNode.sol#L321-L326

`primedBlock` is reset to 0 instead of `block.number` in `StabilizerNode.stabilize`. 
```solidity
    primedBlock = 0; 
```
If `primedBlock` = 0, `block.number > primedBlock + primedWindow` holds in most cases and the next caller of `_validateSwingTraderTrigger` will always get default incentive. But this incenetive is meaningless.
```solidity
      if (block.number > primedBlock + primedWindow) {
        primedBlock = block.number;
        malt.mint(msg.sender, defaultIncentive * (10**malt.decimals()));
        emit MintMalt(defaultIncentive * (10**malt.decimals()));
        return false;
      }
```
So it is recommended to reset `primedBlock` to `block.number` instead of 0.

### [L-03] `skipAuctionThreshold` < `preferAuctionThreshold` should be checked

- https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/StabilizerNode.sol#L359-L370
- https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/StabilizerNode.sol#L437-L439

```solidity
    if (purchaseAmount > preferAuctionThreshold) {
        ...
    } else {
      _startAuction(originalPriceTarget);
    }
```
```solidity
    if (purchaseAmount < skipAuctionThreshold) {
      return;
    }
```    

`skipAuctionThreshold` should be less than `preferAuctionThreshold`.
In `StabilizerNode._triggerSwingTrader`, it starts auction when purchaseAmount <= preferAuctionThreshold.
If `skipAuctionThreshold` >= `preferAuctionThreshold`, `purchaseAmount` <= `skipAuctionThreshold` always holds.
So in `_startAuction`, it will never starts an auction and does nothing. So the `stabilize` will not work in this case. It is recommended to check if `skipAuctionThreshold` < `preferAuctionThreshold` when `skipAuctionThreshold` and `preferAuctionThreshold` are set by the admin.


### [L-04] tradeSize will be only 100%, 50%, 33%, ... because of expansionDampingFactor
- https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/StabilizerNode.sol#L393-L394


```solidity
    uint256 tradeSize = dexHandler.calculateMintingTradeSize(priceTarget) /
      expansionDampingFactor;
```

`tradeSize` will be only 100%, 50%, 33%, ... of minting trade size calculated from `dexHandler`. I think this is intended, but it can be generalized by basis points or 10**18 so it can support other percentages as follows.
```solidity
    uint256 tradeSize = dexHandler.calculateMintingTradeSize(priceTarget) * 10000 / expansionDampingFactorBPS;
```


### [L-05] `updateDesiredAPR` might revert when `aprFloor` < `maxAdjustment`, so aprFloor(2%) must be greater than maxAdjustment(0.5%)

- https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/RewardSystem/RewardThrottle.sol#L131-L183


```solidity
  function updateDesiredAPR() public onlyActive {
    ...
    uint256 newAPR = targetAPR; // gas
    uint256 adjustmentCap = maxAdjustment; // gas
    
    ...

      if (adjustment > adjustmentCap) {
        adjustment = adjustmentCap;
      }

      newAPR -= adjustment;
    }

    uint256 cap = aprCap; // gas
    uint256 floor = aprFloor; // gas
    if (newAPR > cap) {
      newAPR = cap;
    } else if (newAPR < floor) {
      newAPR = floor;
    }

    targetAPR = newAPR;
    aprLastUpdated = block.timestamp;
    emit UpdateDesiredAPR(newAPR);
  }
```

If `aprFloor` < `maxAdjustment`, `newAPR` can be `aprFloor` and `adjustment` can be `maxAdjustment`, so 
`newAPR -= adjustment` will revert. So it needs to make sure that `aprFloor` > `maxAdjustment`.


### [L-06] all balance wasn't sent, some dust would be remained in `_sendToDistributor`

- https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/RewardSystem/RewardThrottle.sol#L124-L128
- https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/RewardSystem/RewardThrottle.sol#L591-L601

In `RewardThrottle.handleReward`, `_sendToDistributor` is called for left balance. 
```solidity
    if (balance > 0) {
      _sendToDistributor(balance, _activeEpoch);
    }

    emit HandleReward(epoch, balance);
```

But in the implementation of `_sendToDistributor`, balance will be splitted to distributors.

```solidity
     uint256 share = (amount * allocations[i]) / 1e18;
     ...
     collateralToken.safeTransfer(distributors[i], share);
```
So some dust will remain in `RewardThrottle` and the actual rewarded amount can be slightly less than balance in `handleReward`. And the event amount(=balance) will be slightly larger than actual rewarded amount

### [L-07] `_triggerSwingTrader` doesn't try `dexHandler.buyMalt` after `swingTraderManager.buyMalt`

- https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/StabilizerNode.sol#L357-L370

`_triggerSwingTrader` doesn't try `dexHandler.buyMalt` after `swingTraderManager.buyMalt`. If capitalUsed is less than purchaseAmount, we can try `dexHandler.buyMalt` with `purchaseAmount` - `capitalUsed`. But current implementation doesn't try `dexHandler.buyMalt` and it misses possible stabilization.

```solidity
    uint256 purchaseAmount = dexHandler.calculateBurningTradeSize(priceTarget);

    if (purchaseAmount > preferAuctionThreshold) {
      uint256 capitalUsed = swingTraderManager.buyMalt(purchaseAmount);

      uint256 callerCut = (capitalUsed * callerRewardCutBps) / 10000;

      if (callerCut != 0) {
        malt.mint(msg.sender, callerCut);
        emit MintMalt(callerCut);
      }
    } else {
      _startAuction(originalPriceTarget);
    }
```

### [L-08] swingTraderManager.getTokenBalances contains inactive swingTrader's balances

- https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/SwingTraderManager.sol#L322-L335

`swingTraderManager.getTokenBalances` doesn't check if swingTrader is active and adds balances regardless the active status.

```solidity
  function getTokenBalances()
    external
    view
    returns (uint256 maltBalance, uint256 collateralBalance)
  {
    uint256[] memory traderIds = activeTraders;
    uint256 length = traderIds.length;

    for (uint256 i; i < length; ++i) {
      SwingTraderData memory trader = swingTraders[activeTraders[i]];
      maltBalance += malt.balanceOf(trader.traderContract);
      collateralBalance += collateralToken.balanceOf(trader.traderContract);
    }
  }
```
But in `buyMalt` and `sellMalt`, they only account balances of active swing traders. This mismatch might cause wrong calculations where `getTokenBalances` are used.


### [L-09] `priceTarget` seems to set to wrong value in `_triggerSwingTrader`

- https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/StabilizerNode.sol#L353-L355

- https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/StabilizerNode.sol#L303-L306

In `StabilizerNode._triggerSwingTrader`, `priceTarget` is set to `icTotal` when `exchangeRate` < `icTotal`.

```solidity
    if (exchangeRate < icTotal) {
      priceTarget = icTotal;
    }
```
If `icTotal` is slightly greater than `exchangeRate`, `priceTarget` can be `exchangeRate` + dust.

But in `_shouldAdjustSupply`, `exchangeRate` should be less than some margin of `priceTarget` to proceed actual stabilization.

```solidity
     return
      (exchangeRate <= (priceTarget - lowerThreshold) &&
        !auction.auctionExists(auction.currentAuctionId())) ||
      exchangeRate >= (priceTarget + upperThreshold);
```
So the `priceTarget` updating logic seems incorrect.


### [N-01] Typo

- https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/StabilizerNode.sol#L411

sandwich is not correct here