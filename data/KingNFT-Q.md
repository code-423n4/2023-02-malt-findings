## Summary
### Low Risk Issues
| Number |Issue Title|Instances|
|:--:|:-------|:--:|
|[L-01]| The ````_removeContract()```` function is not properly implemented  | 1 |
|[L-02]| The ````_updateContract()```` function is not properly implemented | 1 |


Total 2 issues

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
