## Summary
### Low Risk Issues List
| Number |Issue Title|Instances|
|:--:|:-------|:--:|
|[L-01]| [The ````_removeContract()```` function is not properly implemented](#l-01-the-_removecontract-function-is-not-properly-implemented)  | 1 |
|[L-02]| [The ````_updateContract()```` function is not properly implemented](#l-02-the-_updatecontract-function-is-not-properly-implemented) | 1 |
|[L-03]| [The ````updater```` address of ````pool```` may not be updatable](#l-03-the-updater-address-of-pool-may-not-be-updatable) | 1 |
|[L-04]| [The ````ABDKMath64x64```` lib doesn't support tokens with more than 18 decimals](#l-04-the-abdkmath64x64-lib-doesnt-support-tokens-with-more-than-18-decimals) | 1 |

## Low Risk Issues
### [L-01] The ````_removeContract()```` function is not properly implemented
#### Impact
The ````contracts```` array of ````MaltRepository```` contract would be messed up, devs and users would not be able to obtain correct contract string names by the public interface.

```solidity
File: contracts\Repository.sol
26:   string[] public contracts;
```

#### Proof of Concept
As shown of L227 of ````_removeContract()````, the ````currentContract.index```` member variable has been cleard to 0, so local variable ````index```` of L229 will always be 0 too. The ````lastContract```` will be moved to ````contracts[0]````(L230~L231), overwrite the existing first contract name. Meanwhile the desired removing contract name is still in the ````contracts```` array.
```solidity
File: contracts\Repository.sol
223:   function _removeContract(string memory _name) internal {
224:     bytes32 hashedName = keccak256(abi.encodePacked(_name));
225:     Contract storage currentContract = globalContracts[hashedName];
226:     currentContract.contractAddress = address(0);
227:     currentContract.index = 0;
228: 
229:     uint256 index = currentContract.index; // @audit has been overwritten
230:     string memory lastContract = contracts[contracts.length - 1];
231:     contracts[index] = lastContract;
232:     contracts.pop();
233:     emit RemoveContract(hashedName);
234:   }
```


#### Recommended Mitigation Steps
```diff
File: contracts\Repository.sol
223:   function _removeContract(string memory _name) internal {
224:     bytes32 hashedName = keccak256(abi.encodePacked(_name));
225:     Contract storage currentContract = globalContracts[hashedName];
226:     currentContract.contractAddress = address(0);
+        uint256 index = currentContract.index;
227:     currentContract.index = 0;
228: 
- 229:     uint256 index = currentContract.index;
230:     string memory lastContract = contracts[contracts.length - 1];
231:     contracts[index] = lastContract;
232:     contracts.pop();
233:     emit RemoveContract(hashedName);
234:   }
```


### [L-02] The ````_updateContract()```` function is not properly implemented
#### Impact
The data in ````globalContracts```` mapping and ````contracts```` array will be inconsistent.
```solidity
File: contracts\Repository.sol
25:   mapping(bytes32 => Contract) public globalContracts;
26:   string[] public contracts;
```

#### Proof of Concept
As shown of L118\~L123 of ````updateContract()```` and L236\~L242 of ````_updateContract()````, there is no check whether the updating contract has already existed. If ````updateContract```` function is called with non-existing contract parameters, then it will only modify ````globalContracts```` mapping and lead to data inconsistent with ````contracts```` array. In addition ````currentContract.index```` will be incorrect (0), so if there is a subsequent attempt to call ````removeContract()```` to eliminate this mistake operation, then the existing ````contracts[0]```` will be accidentally  removed.
```solidity
File: contracts\Repository.sol
118:   function updateContract(string memory _name, address _contract)
119:     external
120:     onlyRole(TIMELOCK_ROLE)
121:   {
122:     _updateContract(_name, _contract);
123:   }

File: contracts\Repository.sol
236:   function _updateContract(string memory _name, address _newContract) internal {  // @audit contract not existing?
237:     require(_newContract != address(0), "0x0");
238:     bytes32 hashedName = keccak256(abi.encodePacked(_name));
239:     Contract storage currentContract = globalContracts[hashedName];
240:     currentContract.contractAddress = _newContract;
241:     emit UpdateContract(hashedName, _newContract);
242:   }
```

#### Recommended Mitigation Steps
```diff
File: contracts\Repository.sol
236:   function _updateContract(string memory _name, address _newContract) internal {  // @audit contract not existing?
237:     require(_newContract != address(0), "0x0");
238:     bytes32 hashedName = keccak256(abi.encodePacked(_name));
239:     Contract storage currentContract = globalContracts[hashedName];
+        require(currentContract.contractAddress != address(0), "Contract not exists");
240:     currentContract.contractAddress = _newContract;
241:     emit UpdateContract(hashedName, _newContract);
242:   }
```



### [L-03] The ````updater```` address of ````pool```` may not be updatable
#### Impact
As the ````updater```` role is a key role in the protocol, many core contracts requiring the data will not work properly.
```solidity
File: contracts\StabilizedPool\StabilizedPool.sol
40: struct StabilizedPool {
...
47:   address updater;
...
49: }

```

#### Proof of Concept
As shown of L154\~L174 of ````initializeStabilizedPool()````,  there is no check if ````updater != address(0)````.
```solidity
File: contracts\StabilizedPool\StabilizedPoolFactory.sol
154:   function initializeStabilizedPool(
155:     address pool,
156:     string memory name,
157:     address collateralToken,
158:     address updater
159:   ) external onlyRoleMalt(ADMIN_ROLE, "Must have admin role") {
160:     require(pool != address(0), "addr(0)");
161:     require(collateralToken != address(0), "addr(0)");
162:     StabilizedPool storage currentPool = stabilizedPools[pool];
163:     require(currentPool.collateralToken == address(0), "already initialized");
164: 
165:     currentPool.collateralToken = collateralToken;
166:     currentPool.name = name;
167:     currentPool.updater = updater;
168:     currentPool.pool = pool;
169:     _setupRole(POOL_UPDATER_ROLE, updater);
170: 
171:     pools.push(pool);
172: 
173:     emit NewStabilizedPool(pool);
174:   }
```
So the ````updater```` address can be initialized with 0, later admins update it  by calling ````setCurrentPool()````.
However, when we take a closer look at ````setCurrentPool()````, the update can't be done due to the limit on L89.
```solidity
File: contracts\StabilizedPool\StabilizedPoolFactory.sol
079:   function setCurrentPool(address pool, StabilizedPool memory currentPool)
080:     external
081:     onlyRoleMalt(POOL_UPDATER_ROLE, "Must have pool updater role")
082:   {
...
087:     if (
088:       currentPool.updater != address(0) &&
089:       existingPool.updater != address(0) && // @audit can't be updated if existingPool.updater == 0
090:       currentPool.updater != existingPool.updater
091:     ) {
092:       _transferRole(
093:         currentPool.updater,
094:         existingPool.updater,
095:         POOL_UPDATER_ROLE
096:       );
097:       existingPool.updater = currentPool.updater;
098:     }
...
152:   }

```

#### Recommended Mitigation Steps
```diff
File: contracts\StabilizedPool\StabilizedPoolFactory.sol
154:   function initializeStabilizedPool(
155:     address pool,
156:     string memory name,
157:     address collateralToken,
158:     address updater
159:   ) external onlyRoleMalt(ADMIN_ROLE, "Must have admin role") {
160:     require(pool != address(0), "addr(0)");
161:     require(collateralToken != address(0), "addr(0)");
162:     StabilizedPool storage currentPool = stabilizedPools[pool];
163:     require(currentPool.collateralToken == address(0), "already initialized");
+        require(updater != address(0), "addr(0)");
164: 
...
174:   }

```

### [L-04] The ````ABDKMath64x64```` lib doesn't support tokens with more than 18 decimals
#### Impact
The following functions of ````MaltDataLab```` contract will always revert if ````collateralToken````'s decimals is more than 18.
```solidity
  function getRealBurnBudget(uint256 maxBurnSpend, uint256 premiumExcess);
  function getSwingTraderEntryPrice();
  function getActualPriceTarget();
```
As these functions play core roles in the system, so tokens with more than 18 decimals, as an example YAMv2 has 24 decimals, will not work.
#### Proof of Concept
As shown on L63 of ````ABDKMath64x64.sol````, the ````fromUInt()```` function requires the ````x```` parameter less than ````0x7FFFFFFFFFFFFFFF```` which is about ````9e18````.
```solidity
File: contracts\libraries\ABDKMath64x64.sol
61:   function fromUInt(uint256 x) internal pure returns (int128) {
62:     unchecked {
63:       require(x <= 0x7FFFFFFFFFFFFFFF); // @audit 0x7FFFFFFFFFFFFFFF = 9,223,372,036,854,775,807
64:       return int128(int256(x << 64));
65:     }
66:   }
```
So, if ````collateralToken````'s decimals ````> 18````, then the following lines of ````MaltDataLab```` contract will revert, L240 of ````getRealBurnBudget()```` function
```solidity
File: contracts\DataFeed\MaltDataLab.sol
230:   function getRealBurnBudget(uint256 maxBurnSpend, uint256 premiumExcess)
...
234:   {
...
237: 
238:       int128 stMaltRatioInt = ABDKMath64x64
239:         .fromUInt(swingTraderManager.calculateSwingTraderMaltRatio())
240:         .div(ABDKMath64x64.fromUInt(10**collateralToken.decimals())) // @audit revert
241:         .mul(ABDKMath64x64.fromUInt(100));
...
258:   }

```
L389 of ````getSwingTraderEntryPrice()```` function
```solidity
 File: contracts\DataFeed\MaltDataLab.sol
350:   function getSwingTraderEntryPrice()
...
354:   {
...
370:     uint256 unity = 10**collateralToken.decimals();
...
389:     int128 unityInt = ABDKMath64x64.fromUInt(unity); // @audit revert
...
429:   }

```
L442 of ````getActualPriceTarget()```` function
```solidity
File: contracts\DataFeed\MaltDataLab.sol
431:   function getActualPriceTarget() external view returns (uint256) {
432:     uint256 unity = 10**collateralToken.decimals();
...
442:     int128 unityInt = ABDKMath64x64.fromUInt(unity);
...
482:   }

```

#### Recommended Mitigation Steps
Using more precise math such as ````128x128````.
