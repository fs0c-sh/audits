## [M-01] Liquidation incentives not in favour of system health.

### Impact

The liquidations would not be in favour of overall system health and some lower ICR troves could remain unliquidated, and will eventually cross the 100% collateralization rate line where liquidating them results in a loss for other system participants.

### Proof of Concept

The liquidations can be carried out via two different methods, either liquidating n number of troves (which will liquidate the troves starting from the one with the lowest collateral ratio in the system) and batch liquidations (where a custom list of troves can be given).

The liquidator’s compensation depends on this function:

```solidity
function _getCollGasCompensation(uint _entireColl) internal pure returns (uint) {
        return _entireColl / PERCENT_DIVISOR;
    }
```

That is, the 0.5% of the total collateral of the liquidated trove. Which means that liquidators are incentivized to liquidate troves with more collateral. If these troves consist of troves with ICR close to TCR (more collateral and less debt) then the impact on TCR will be less that the impact from liquidating troves with smaller ICR.

**As a consequence, liquidators have more incentive to liquidate troves that have less impact on the overall health of the system as measured by the TCR.**

During high crypto volatility, automated bots will liquidate those troves that generate more profit for them, but are not optimal for the system's health. Those lower ICR troves will remain unliquidated, and will eventually cross the 100% collateralization rate line where liquidating them results in a loss for other system participants.

### Tools Used

Manual Analysis

### Recommendations

One way to solve this can be to liquidate some percentage of the lowest collateral ratio troves along with the selected troves in batchliquidations. So the liquidators are forced to liquidate some of the low ICR troves along with the troves they wish to liquidate. This way the overall system can be healthy.

## [M-01] **High-fraction liquidations cause the global running product `P` to become less than `1e9`.**

### Impact

The global variable `P` can be made smaller than `1e9` via repeated liquidations that liquidate a high fraction of the SP. When `P` becomes small enough, further liquidations can cause `newP` to evaluate to `0` in `_updateRewardSumAndProduct`.

The transaction then reverts due to the assert on `assert(newP > 0)`.

In this state, further high-fraction liquidations would be blocked.

### Proof of Concept

An implicit assumption was that `P` never goes below `1e9`, since at `P <1e9`, the scale changes and we multiply `P` by `1e9` . However depletions can leave the pool with much lesser `P` and in those cases multiplying to `1e9` is not always the solution.

The attack would be a 3-liquidation sequence whereby each liquidation reduces the SP to the smallest possible fraction of its prior value, i.e. 1e-18.

Here, starting with the “best” value of P (P = 1e18), then after 2 liquidations, P = 1, and further high-fraction liquidations revert.

```solidity
To calculate newProductFactor we need:
    
    _LUSDLossPerUnitStaked = _debtToOffset * 1e18 / _totalLUSDDeposits + 1
    We need _LUSDLossPerUnitStaked to be 1e18 - 1. So:
    1e18 - 1 = _debtToOffset * 1e18 / _totalLUSDDeposits + 1
    1e18 - 2 = _debtToOffset * 1e18 / _totalLUSDDeposits
    _totalLUSDDeposits * (1e18 - 2) <= _debtToOffset * 1e18 < _totalLUSDDeposits * (1e18 - 1)
    Assuming _totalLUSDDeposits = 10,000e18
    10000e18 * (1e18 - 2) <= _debtToOffset * 1e18 < 10000e18 * (1e18 - 1)
    10000e18 * 999,999,999,999,999,998 <= _debtToOffset * 1e18 < 10000e18 * 999,999,999,999,999,999
    9999999999999999980000 <= _debtToOffset < 9999999999999999990000
    Translated to decimal notation:
    9999.999999999999980000 <= _debtToOffset < 9999.999999999999990000
    
    - Start:
    
    P = 1e18
    Deposits = 10,000e18
    
    - Liquidation 1:
    
    Burn: 9,999,999,999,999,999,980,000
    NewP = 1e18 * 1 * 1e9 / 1e18 = 1e9
    
    - Liquidation 2:
    
    New deposit: 9,999,999,999,999,999,980,000
    Total deposit = 10,000e18
    
    Burn: 9,999,999,999,999,999,980,000
    NewP = 1e9 * 1 * 1e9 / 1e18 = 1
    
    - Liquidation 3:
    
    New deposit: 9,999,999,999,999,999,980,000
    Total deposit = 10,000e18
    
    Burn: 9,999,999,999,999,999,980,000
    NewP = 1 * 1 * 1e9 / 1e18 = 0
```

### Recommendations

The following change can be made to the function:

```solidity
newP = currentP.mul(newProductFactor).mul(SCALE_FACTOR).div(DECIMAL_PRECISION); 
             currentScale = currentScaleCached.add(1);
             emit ScaleUpdated(currentScale);
+            // If it’s still smaller than the SCALE_FACTOR, increment scale again. Afterwards it couldn’t happen again, as DECIMAL_PRECISION = SCALE_FACTOR^2
+            if (currentP.mul(newProductFactor).div(DECIMAL_PRECISION) < SCALE_FACTOR) {
+                newP = newP.mul(SCALE_FACTOR);
+                currentScale = currentScaleCached.add(1);
+                emit ScaleUpdated(currentScale);
+            }
         } else {
             newP = currentP.mul(newProductFactor).div(DECIMAL_PRECISION);
         }
```