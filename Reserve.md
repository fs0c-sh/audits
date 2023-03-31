## [M-01 ] If name is changed then the domain seperator would be wrong.

### Vulnerability Details

In `StRSR.sol` the `_domainSeparatorV4` is calculated using the EIP-721 standard, which uses the `name` and `version` that are passed in the init at the function call `__EIP712_init(name, "1");`

Now, governance can change this `name` anytime using the following function:

```solidity
function setName(string calldata name_) external governance {
        name = name_;
    }
```

After that call the domain seperator would still be calculated using the old name, which shouldn’t be the case. 

### Impact

The permit transactions would be reverted if the domain seperator is wrong. 

### Recommendation

While changing the name in in setName function. update the domain seperator.

## [H-02] Token with double entry points can be used.

### Impact

Some tokens can have two entry points ,i.e., two different contracts that both interact with same balances. Example TUSD, the real implementation of TUSD is at the address `0x0000000000085d4780b73119b644ae5ecd22b376` , and the secondary entry point `0x8dd5fbce2f6a956c3022ba3663759011dd51e73e`simply forwards any calls to the primary contract.

In the contract `BackingManager.sol` the functions to manage tokens do not take these tokens into account. eg:

```solidity
function manageTokens(IERC20[] calldata erc20s) external notPausedOrFrozen {
        // Token list must not contain duplicates
        require(ArrayLib.allUnique(erc20s), "duplicate tokens");
        _manageTokens(erc20s);
    }
```

The require statement is not enough to check if tokens with two entry points are used as the addresses would be different. The `_manageTokens` later calls `handoutExcessAssets` which would cause the assets to be sent twice. In future there might be more tokens out there with multiple entry points. For example, Tether USD (USDT) could enable a secondary entry point in the future when upgrading their contract.

### POC

```solidity
function manageTokensSortedOrder(IERC20[] calldata erc20s) external notPausedOrFrozen {
        // Token list must not contain duplicates
        require(ArrayLib.sortedAndAllUnique(erc20s), "duplicate/unsorted tokens");
        _manageTokens(erc20s);
    }

function manageTokens(IERC20[] calldata erc20s) external notPausedOrFrozen {
        // Token list must not contain duplicates
        require(ArrayLib.allUnique(erc20s), "duplicate tokens");
        _manageTokens(erc20s);
    }
```

### Recommendation

A blacklist can be kept for such tokens, to revert if any such tokens is used.