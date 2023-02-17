## Summary
### Low Risk Issues List
| Number |Issue Title|Instances|
|:--:|:-------|:--:|
|[L-01]| [The ````_removeContract()```` function is not properly implemented](#l-01-the-_removecontract-function-is-not-properly-implemented)  | 1 |
|[L-02]| [The ````_updateContract()```` function is not properly implemented](#l-02-the-_updatecontract-function-is-not-properly-implemented) | 1 |
|[L-03]| [The ````updater```` address of ````pool```` may not be updatable](#l-03-the-updater-address-of-pool-may-not-be-updatable) | 1 |

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
