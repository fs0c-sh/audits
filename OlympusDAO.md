## [M-01] `withdrawAndUnwrap` boolean return value not handled

## Summary

The return value of `withdrawAndUnwrap` call on auraPool is not handled and might silently fail.

## Vulnerability Detail

When the `_withdraw` function is called, the following call is made to the auraPool:

```jsx
auraPool.rewardsPool.withdrawAndUnwrap(lpAmount_, false);
```

The return value of this call in unhandled. This call might silently fail. The actual call to  `withdrawAndUnwrap` can be seen here [https://github.com/convex-eth/platform/blob/ece5998c54b0354a60f092e0dda1aa1f040ec8bd/contracts/contracts/BaseRewardPool.sol#L238](https://github.com/convex-eth/platform/blob/ece5998c54b0354a60f092e0dda1aa1f040ec8bd/contracts/contracts/BaseRewardPool.sol#L238) . Here the function returns a boolean value:

```jsx
function withdrawAndUnwrap(uint256 amount, bool claim) public updateReward(msg.sender) returns(bool){
```

There is another place with similar vulnerability, its in the function `rescueFundsFromAura` will similar unhandled value.

## Impact

Without handling return value the transaction might silently fail. Note, that the interface defined as `IAuraRewardPool` might also needed to be changed to include those boolean values. 

## Code Snippet

[https://github.com/sherlock-audit/2023-02-olympus-fs0c-sh/blob/main/src/policies/lending/WstethLiquidityVault.sol#L175](https://github.com/sherlock-audit/2023-02-olympus-fs0c-sh/blob/main/src/policies/lending/WstethLiquidityVault.sol#L175)

[https://github.com/sherlock-audit/2023-02-olympus-fs0c-sh/blob/main/src/policies/lending/WstethLiquidityVault.sol#L357](https://github.com/sherlock-audit/2023-02-olympus-fs0c-sh/blob/main/src/policies/lending/WstethLiquidityVault.sol#L357)

## Tool used

Manual Analysis

> References:
> 

[https://github.com/sherlock-audit/2022-09-notional-judging/issues/118](https://github.com/sherlock-audit/2022-09-notional-judging/issues/118)

## Recommendation

Handle those return values

```solidity
bool unstaked = auraPool.rewardsPool.withdrawAndUnwrap(lpAmount_, false);
require(unstaked, 'unstake failed');
```

## [H-01] Withdraw transactions would revert if ohmReceived > ohmMinted.

## Summary

The withdraw transaction might get reverted if ohmReceived from the `_withdraw` function call is greater than the ohmMinted. This might happen for early/first investor if the price of LP is increased and they would want to withdraw their Liquidity. 

## Vulnerability Detail

A user calls `withdraw` with the LP amount they want to burn and they would received the pairToken based on the amount of LP token. This LP token would be then used to calculate the amount of ohm tokens received and the pair tokens received. 

Following the the withdraw function:

```solidity
function withdraw(
        uint256 lpAmount_,
        uint256[] calldata minTokenAmounts_,
        bool claim_
    ) external onlyWhileActive nonReentrant returns (uint256) {
        // Liquidity vaults should always be built around a two token pool so we can assume
        // the array will always have two elements
        if (lpAmount_ == 0 || minTokenAmounts_[0] == 0 || minTokenAmounts_[1] == 0)
            revert LiquidityVault_InvalidParams();
        if (!_isPoolSafe()) revert LiquidityVault_PoolImbalanced();

        _withdrawUpdateRewardState(lpAmount_, claim_);

        totalLP -= lpAmount_;
        lpPositions[msg.sender] -= lpAmount_;

        // Withdraw OHM and pairToken from LP
        (uint256 ohmReceived, uint256 pairTokenReceived) = _withdraw(lpAmount_, minTokenAmounts_);

        // Reduce deposit values
        uint256 userDeposit = pairTokenDeposits[msg.sender];
        pairTokenDeposits[msg.sender] -= pairTokenReceived > userDeposit
            ? userDeposit
            : pairTokenReceived;
        ohmMinted -= ohmReceived > ohmMinted ? ohmMinted : ohmReceived;
        ohmRemoved += ohmReceived > ohmMinted ? ohmReceived - ohmMinted : 0;

        // Return assets
        _repay(ohmReceived);
        pairToken.safeTransfer(msg.sender, pairTokenReceived);

        emit Withdraw(msg.sender, pairTokenReceived, ohmReceived);
        return pairTokenReceived;
    }
```

Assume a scenario where the first user deposits some LP token and the price of LPs increased. They would call `withdraw` for the withdrawal, which will call `_withdraw` internally. This function would return the ohmReceived value, and as the LP price is increased, the ohmReceived would be greater than the ohmMinted in the deposit call. 

In the above function the value of `ohmMinted` would now become 0. The value of `ohmRemoved` would be `ohmReceived` and the `_repay` would be called with the `ohmReceived` value. 

This `_repay` function would internally call `_burn` on `OlympusERC20` defined as follows:

```solidity
function _burn(address account, uint256 amount) internal virtual {
        require(account != address(0), "ERC20: burn from the zero address");

        _beforeTokenTransfer(account, address(0), amount);

        _balances[account] = _balances[account].sub(amount, "ERC20: burn amount exceeds balance");
        _totalSupply = _totalSupply.sub(amount);
        emit Transfer(account, address(0), amount);
    }
```

As the balances[vault] would be less than the `amount` because in the deposit call the minted value would be less than `ohmReceived` this subtraction would revert. 

## Impact

User would not be able to withdraw their tokens. The withdraw would work if the prices is dropped again or if other users deposit in the vault, this would create some loss for the user as they wanted to withdraw when the prices were in their favour.

## Code Snippet

[https://github.com/sherlock-audit/2023-02-olympus-fs0c-sh/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L280](https://github.com/sherlock-audit/2023-02-olympus-fs0c-sh/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L280)

## Tool used

Manual Review

## Recommendation

`_repay` should use the `ohmRemoved` value, as that is the actual value of ohm Removed. 

## [H-01] Users can claim rewards multiple times.

## Summary

Users can claim rewards multiple times because the userRewardDebts is wrongly set to zero before subtracting from cachedUserRewards.

## Vulnerability Detail

The value of `cachedUserRewards` is wrongly updated here, in the `_withdrawUpdateRewardState`:

```solidity
if (rewardDebtDiff > userRewardDebts[msg.sender][rewardToken.token]) {
                userRewardDebts[msg.sender][rewardToken.token] = 0;
                cachedUserRewards[msg.sender][rewardToken.token] +=
                    rewardDebtDiff -
                    userRewardDebts[msg.sender][rewardToken.token]; //@audit this would always be zero?
            } else {
                userRewardDebts[msg.sender][rewardToken.token] -= rewardDebtDiff;
            }
```

The value of `userRewardDebts[msg.sender][rewardToken.token]` is subtracted from rewardDebtDiff, but it has being zeroed out in the previous line. Which means the value of `cachedUserRewards[msg.sender][rewardToken.token]` would be `cachedUserRewards[msg.sender][rewardToken.token] + rewardDebtDiff` . This means that the `userRewardDebts` value would not be subtracted from it.

The attack path is as follows:

- Call `claimRewards` first. This will first call `_accumulateExternalRewards` to accumulate rewards first and then update external rewards state and then call `_claimExternalRewards` to claim the rewards. Here the value of `userRewardDebts[msg.sender][rewardToken.token]` would be updated to add the rewards that user claimed.
- Next call `withdraw` with `claim_` set as `false`. This would internally call `_withdrawUpdateRewardState` and here because of the bug the value of `cachedUserRewards[msg.sender][rewardToken.token]` would be wrongly updated without taking into consideration the rewards that user already claimed. This value would be updated to `lpAmount_ * rewardToken.accumulatedRewardsPerShare` .
- In the next step user can call `claimRewards` again and the function `externalRewardsForToken` called from `_claimExternalRewards` would return the value of `cachedUserRewards[user_][rewardToken.token] + totalAccumulatedRewards` which can be claimed again.

## Impact

User can claim rewards multiple times.

## Code Snippet

[https://github.com/sherlock-audit/2023-02-olympus-fs0c-sh/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L584](https://github.com/sherlock-audit/2023-02-olympus-fs0c-sh/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L584)

[https://github.com/sherlock-audit/2023-02-olympus-fs0c-sh/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L607](https://github.com/sherlock-audit/2023-02-olympus-fs0c-sh/blob/main/src/policies/lending/abstracts/SingleSidedLiquidityVault.sol#L607)

## Tool used

Manual Review

## Recommendation

Move the line `userRewardDebts[msg.sender][rewardToken.token] = 0;` after this line:

```solidity
cachedUserRewards[msg.sender][rewardToken.token] +=
                    rewardDebtDiff -
                    userRewardDebts[msg.sender][rewardToken.token];
```