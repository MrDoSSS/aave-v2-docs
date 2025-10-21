# Risk Parameters & Interest Model

## Context
Risk configuration in Aave v2 is managed per reserve via the `ReserveConfigurationMap` and enforced during validation. Interest rates are derived from utilisation metrics using an adjustable, two-slope model implemented in the reserve’s interest rate strategy contract.

## Reserve Risk Parameters

| Field | Type | Applies to | Description |
|-------|------|------------|-------------|
| `ltv` | bps | Collateral asset | Maximum percentage of collateral value that can be borrowed. Determines borrow power while staying above liquidation. |
| `liquidationThreshold` | bps | Collateral asset | Level at which health factor reaches 1.0 and liquidation becomes possible. Always ≥ LTV. |
| `liquidationBonus` | bps | Collateral asset | Extra collateral awarded to liquidators over the repaid debt to compensate slippage (e.g., 10500 = 5% bonus). |
| `reserveFactor` | bps | Reserve | Fraction of interest accrued that is redirected to the protocol treasury instead of depositors. |
| `borrowingEnabled` | bool | Reserve | If false, disables new borrows (stable and variable) for the asset. |
| `stableBorrowRateEnabled` | bool | Reserve | If false, users can only open variable-rate borrows on the asset. |
| `collateralEnabled` | bool | Reserve | Controls whether the asset can be used as collateral (set by configuring LTV/LT/bonus to non-zero values). |
| `pause` / `freeze` | bool | Pool / Reserve | Safety switches. Pause stops all pool actions; freeze halts new supply/borrow/rate swaps on a specific reserve while allowing repayments and liquidations. |
| `supplyCap` / `borrowCap` | n/a | V3 feature | Not present in Aave v2. Mentioned here for clarity; any caps must be enforced off-protocol. |

## Interest Rate Strategy Parameters

| Field | Type | Description |
|-------|------|-------------|
| `optimalUtilizationRate` | bps | Utilisation at which the rate curve kinks (i.e., 80% = 0.8). |
| `baseVariableBorrowRate` | ray | Variable borrow rate when utilisation is zero. |
| `variableRateSlope1` | ray | Incremental rate applied as utilisation rises from 0 to the optimal utilisation. |
| `variableRateSlope2` | ray | Additional slope applied when utilisation exceeds the optimal rate, sharply increasing borrowing costs. |
| `stableRateSlope1` | ray | Stable borrow slope below optimal utilisation; added to base stable rate fetched from the lending rate oracle. |
| `stableRateSlope2` | ray | Stable borrow slope above optimal utilisation; penalises stressed markets. |
| `reserveFactor` | bps | Discount applied to liquidity rate calculation to divert part of interest to the treasury. |

### Utilisation and Rate Formulas

Let:
- `U = totalDebt / (availableLiquidity + totalDebt)`
- `U_opt = optimalUtilizationRate / 10_000`
- Quantities in ray unless noted.

Variable rate:
```
if U <= U_opt:
    r_var = baseVariableBorrowRate + variableRateSlope1 * (U / U_opt)
else:
    excess = (U - U_opt) / (1 - U_opt)
    r_var = baseVariableBorrowRate + variableRateSlope1 + variableRateSlope2 * excess
```

Stable rate (with lending-rate oracle base `r_oracle`):
```
if U <= U_opt:
    r_stable = r_oracle + stableRateSlope1 * (U / U_opt)
else:
    excess = (U - U_opt) / (1 - U_opt)
    r_stable = r_oracle + stableRateSlope1 + stableRateSlope2 * excess
```

Liquidity rate paid to depositors:
```
r_liquidity = (r_weighted_borrow * U) * (1 - reserveFactor)
```
where `r_weighted_borrow` is the utilisation-weighted average of variable and stable borrow rates.

All rates are annualised; conversions between ray and APR use `rate / 1e27`. When integrating, respect rounding towards zero to match Solidity behaviour (`WadRayMath`).***
