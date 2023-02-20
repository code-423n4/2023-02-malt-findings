# Summary

| Id | Title |
| -- | ----- |
| 1 | Function `updateDesiredAPR()` might revert when does the `newAPR -= adjustment` calculation |
| 2 | Event `SellMalt()` is not emitted when dust check is passed |
| 3 | Protocol did not work with collateral token has value different than $1 |
| 4 | Open TODOs |
| 5 | Duplicated `_swingTrader` addresses can be added which make `sellMalt()/buyMalt()` working incorrectly |  

# 1. Function `updateDesiredAPR()` might revert when does the `newAPR -= adjustment` calculation

https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/RewardSystem/RewardThrottle.sol#L169

## Detail
When admin set variables for `proportionalGainBps`, `cushionBps`, there are some cases `adjustment > newAPR` and make the calculation revert. `newAPR` is capped at floor, so it should be at least `floor` even when the result of calculation smaller than 0.

## Recommendation
Consider adding a check to ensure the result will be at least `floor`. For example `if (newAPR < adjustment) newAPR = floor;`


# 2. Event SellMalt is not emitted when dust check is passed

https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/SwingTraderManager.sol#L258

## Detail
This is similar to the issue where `totalProfit` value is updated incorrectly. In this case, event `SellMalt` should be emitted before the dust check to ensure it will always be emitted.

## Recommendation
Consider moving the event emission before the dust check.

# 3. Protocol did not work with collateral token has value different than $1

## Detail
There are multiple places in the protocol that assume the collateral token has value of `$1` even though it has a `priceTarget` variable that admin can set. 

For example, when compare `actualTarget` with `localTarget` in function `getActualPriceTarget()`. `actualTarget` is not normalized to `localTarget` so they should not be able to compare with each other. It can be compared only when `localTarget` is assumed to has value `1e18` (collateral token has 18 decimals and has value $1)

https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/DataFeed/MaltDataLab.sol#L475-L479

```solidity
if (normActualTarget > localTarget) {
  return localTarget;
} else if (actualTarget < icTotal && icTotal < localTarget) {
    // @audit actualTarget is not the same unit with localTarget
  return (icTotal * localTarget) / unity;
}
```

So for now, if admin set `priceTarget != 1`, the protocol will not work as expected (normalize price based on priceTarget).

# 4. Open TODOs

Code architecture, incentives, and error handling/reporting questions/issues should be resolved before deployment

## Instances
https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/StabilizerNode.sol#L352-L355

```solidity
// TODO StabilizerNode.sol these checks won't work when working with pools not pegged to 1 Wed 26 Oct 2022 16:40:25 BST
    if (exchangeRate < icTotal) {
      priceTarget = icTotal;
    }
```

https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/StabilizerNode.sol#L441-L442

```solidity
// TODO StabilizerNode.sol invert priceTarget? Fri 21 Oct 2022 11:02:43 BST
    bool success = auction.triggerAuction(priceTarget, purchaseAmount);
```

# 5. Duplicated `_swingTrader` addresses can be added which make `sellMalt()/buyMalt()` working incorrectly

https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/SwingTraderManager.sol#L407

## Details
In function `addSwingTrader()`, there is no check to ensure `_swingTrader` address is not existed. So admin can make a mistake and add the same `_swingTrader` address twice.

As the results, when there are duplicated `_swingTrader` addresses, all the for-loop through the swing trader lists will accounts the same address twice and lead to wrong result. It will affect `sellMalt()` and `buyMalt()` which are core functions of the contract.

This issue depends on admin to make the mistake but it is always better to add a input check. 

## Recommendation
Consider adding check to ensure there is no duplicated `traderContract` addresses can be added.
