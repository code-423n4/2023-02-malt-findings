# Summary

| Id | Title |
| -- | ----- |
| 1 | Unnecessary external call in `calculateSwingTraderMaltRatio()` |
| 2 | Avoid use storage variable in event emission |

# 1. Unnecessary external call in `calculateSwingTraderMaltRatio()`

https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/StabilityPod/SwingTraderManager.sol#L296

## Detail
In function `calculateSwingTraderMaltRatio()`, `maltDecimals` is get using an external call to Malt contract. However, it did not use anywhere in the function.
```solidity
uint256 maltDecimals = malt.decimals(); // @audit gas unnecessary get
```

## Recommendation
Removing unnecessary external call to save gas.


# 2. Avoid use storage variable whenever possible to save gas

https://github.com/code-423n4/2023-02-malt/blob/700f9b468f9cf8c9c5cffaa1eba1b8dea40503f9/contracts/Token/TransferService.sol#L205

## Detail
```solidity
function acceptVerifierManagerRole() external {
  require(msg.sender == proposedManager, "Must be proposedManager");
  _transferRole(proposedManager, verifierManager, VERIFIER_MANAGER_ROLE); 
  // @audit use msg.sender instead of proposedManager
  proposedManager = address(0);
  verifierManager = msg.sender;
  emit ChangeVerifierManager(msg.sender);
}
```

In call `_transferRole()`, we can use `msg.sender` instead of `proposedManager` to save gas since they are validated to have the same value in above require check.

## Recommendation
Consider using `msg.sender` instead of `proposedManager`.
