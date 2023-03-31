## [M-01] Small precision loss in splits functionality can lead to unequal distribution of funds between users with same `splitsWeight`.

### Impact

There is a small precision loss in splits functionality which can cause unfair splits between users with same `splitsWeight` . Under normal circumstances such precision could be ignored (Eg: if all the users suffered the small loss), but in this case a few users get less funds than the other users due to which it causes an unfair split and seems like a legit issue to me.

### POC

The `_split` function in Splits.sol:

```solidity
function _split(uint256 userId, uint256 assetId, SplitsReceiver[] memory currReceivers)
        internal
        returns (uint128 collectableAmt, uint128 splitAmt)
    {
        _assertCurrSplits(userId, currReceivers);
        mapping(uint256 => SplitsState) storage splitsStates = _splitsStorage().splitsStates;
        SplitsBalance storage balance = splitsStates[userId].balances[assetId];

        collectableAmt = balance.splittable;
        if (collectableAmt == 0) {
            return (0, 0);
        }

        balance.splittable = 0;
        uint32 splitsWeight = 0;
        for (uint256 i = 0; i < currReceivers.length; i++) {
            splitsWeight += currReceivers[i].weight;
            uint128 currSplitAmt =
                uint128((uint160(collectableAmt) * splitsWeight) / _TOTAL_SPLITS_WEIGHT - splitAmt);
            splitAmt += currSplitAmt;
            uint256 receiver = currReceivers[i].userId;
            _addSplittable(receiver, assetId, currSplitAmt);
            emit Split(userId, receiver, assetId, currSplitAmt);
        }
        collectableAmt -= splitAmt;
        balance.collectable += collectableAmt;
        emit Collectable(userId, assetId, collectableAmt);
    }
```

The current split amount is calculated by adding all the weight including the current weight , multiplying the total amount and subtracting the amount splitted till now.

Assume a case where the value of `balance.splittable = 5` and two weights of `30%` and `30%` are used. 

The value of `currSplitAmt` in first itteration would be: 5 * 300_000 / 1_000_000 = 1

, and in the second itteration would be: 5 * 600_000 / 1_000_000 - 1 = 2

So, the first user got `1` token while the second user got `2` tokens for the same percentage of split. This causes unfair split.

A POC in foundry with `2555555` initial value and 200 receivers with 0.5% split is as follows, it causes unfair split for **30% of the users** :

```solidity
function testCollect() public {
        uint128 amt = 2555555;
        
        SplitsReceiver[] memory sr = new SplitsReceiver[](200);
        for(uint i=0;i<200;i++){
            sr[i].userId = uint256((10)*(i+1));
            sr[i].weight = 5_000;
        }
        
        driver.give(tokenId1, tokenId2, erc20, amt);
        driver.setSplits(tokenId2,sr);
        dripsHub.split(tokenId2, erc20, sr);
        for(uint i=0;i<200;i++){
            emit log_uint(dripsHub.splittable(uint256((10)*(i+1)) , erc20 ));
        }
}
```

The disproportion can be increased in case of tokens with less decimal places such as `0xdb25f211ab05b1c97d595516f45794528a807ad8` (2 decimal places) or tokens with high value such as WBTC(8 decimal), which in this case causes unfair difference of 0.02$ for each user at the current price of BTC.

### Recommendation Mitigation Steps

Rewriting the function to the following should be correct:

```solidity
function _split( userId, uint256 assetId, SplitsReceiver[] memory currReceivers)
        internal
        returns (uint128 collectableAmt, uint128 splitAmt)
    {
        _assertCurrSplits(userId, currReceivers);
        mapping(uint256 => SplitsState) storage splitsStates = _splitsStorage().splitsStates;
        SplitsBalance storage balance = splitsStates[userId].balances[assetId];

        collectableAmt = balance.splittable;
        if (collectableAmt == 0) {
            return (0, 0);
        }

        balance.splittable = 0;
        uint32  = 0;
        for (uint256 i = 0; i < currReceivers.length; i++) {
            uint128 currSplitAmt =
                uint128((uint160(collectableAmt) * currReceivers[i].weight) / _TOTAL_SPLITS_WEIGHT);
            splitAmt += currSplitAmt;
            uint256 receiver = currReceivers[i].userId;
            _addSplittable(receiver, assetId, currSplitAmt);
            emit Split(userId, receiver, assetId, currSplitAmt);
        }
        collectableAmt -= splitAmt;
        balance.collectable += collectableAmt;
        emit Collectable(userId, assetId, collectableAmt);
    }
```

## [G-01] In the splits functionality an extra variable declaration can be reduced.

Current Implementation:

```solidity
function _split(uint256 userId, uint256 assetId, SplitsReceiver[] memory currReceivers)
        internal
        returns (uint128 collectableAmt, uint128 splitAmt)
    {
        _assertCurrSplits(userId, currReceivers);
        mapping(uint256 => SplitsState) storage splitsStates = _splitsStorage().splitsStates;
        SplitsBalance storage balance = splitsStates[userId].balances[assetId];
```

Recommended:

```solidity
function _split(uint256 userId, uint256 assetId, SplitsReceiver[] memory currReceivers)
        internal
        returns (uint128 collectableAmt, uint128 splitAmt)
    {
        _assertCurrSplits(userId, currReceivers);
        SplitsBalance storage balance = _splitsStorage().splitsStates[userId].balances[assetId];
```

Gas usage before in calling `testCollectTransfersFundsToTheProvidedAddress:NFTDriver.t.sol` :207571 

Gas usage after in calling `testCollectTransfersFundsToTheProvidedAddress:NFTDriver.t.sol` :207567

## [QA-01] An authorized user can authorize/unauthorize other users.

### Impact

The caller.sol adds a functionality for a user to authorize other users to make a call on behalf on them. This is done by calling the `authorize` function from the user who want’s to authorize other users with the address of the user they want to authorize as the parameter to the function.

Let’s say a user wants to authorize 2 users A and B to make calls on behalf of them. Both the users should be at same priviledge level and user A should not be able to unauthorize user B from its priviledge. 

This vulnerability allows user A to perfrom such actions and unauthorize user B. This can also be used to unauthorize by front-running a legit transaction of user B. 

> Note: I’ve asked the team, and they consider this is not an issue, that’s why I am reported it as QA as it still feels like a priviledge issue to me and I am keeping this open to discussion.
> 

### POC

Add the following lines in `AddressDriver.t.sol`

```solidity
address internal user2 = address(1337);

function testCanunauthorizeotherusers() public {
        vm.prank(user);
        caller.authorize(address(this));
        vm.prank(user);
        caller.authorize(user2);

        bool resultfirst = caller.isAuthorized(user, user2);
        assertTrue(resultfirst, "is unauthorized");

        bytes memory callData = abi.encodeWithSelector(caller.unauthorize.selector, user2);
        caller.callAs(user, address(caller), callData);

        bool resultsecond = caller.isAuthorized(user, user2);
        assertFalse(resultsecond, "is authorized");
    }
```

### Recommendation

Only allow necessary functions to be called by authorized users.