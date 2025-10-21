# Health Factor & Liquidation Mechanics

## Context
Aave v2 continuously evaluates the solvency of each user account with the **health factor (HF)**. When HF drops below 1, the position becomes liquidatable. This section formalises the calculation, highlights key parameters, and walks through a numerical example.

## Definitions
- **`LTV` (Loan-to-Value):** Maximum borrowing power for an asset, expressed in basis points.
- **`Liquidation Threshold (LT)`:** Collateral value weighting applied during liquidation checks; when the weighted collateral no longer covers debt, HF ≤ 1.
- **`Liquidation Bonus`:** Percentage premium granted to liquidators on top of repaid debt.
- **`Close Factor`:** Maximum portion of outstanding debt that can be repaid in a single liquidation call. In v2 this is hard-coded to 50% (5_000 bps) in `LendingPoolCollateralManager`.

## Health Factor Formula

Collateral and debt values are measured in ETH using the price oracle. Let:
- `collateralValue = Σ_i (balance_i × price_i)`
- `weightedCollateral = Σ_i (balance_i × price_i × LT_i / 10_000)`
- `debtValue = Σ_j (debt_j × price_j)`

Then:
```
HF = weightedCollateral / debtValue
```

Equivalent form from protocol source:
```
HF = (Σ(collateral_i_value × LT_i) / Σ(collateral_i_value)) × (Σ(collateral_i_value) / Σ(debt_j_value))
```

- If `HF > 1`, the position is solvent.
- If `HF <= 1`, any address may trigger `liquidationCall`.

## Worked Example

| Parameter | Value |
|-----------|-------|
| Collateral | 100 ETH worth of DAI (LT = 82.5%) |
| Debt | 70 ETH worth of USDC |

Initial HF:
```
HF = (100 × 0.825) / 70 = 1.178  → safe
```

20% price drop on collateral (80 ETH):
```
HF = (80 × 0.825) / 70 = 0.942  → liquidatable
```

## Liquidation Flow Highlights
1. **Trigger:** Liquidator calls `liquidationCall` specifying collateral asset, debt asset, user, and desired debt to cover.
2. **Validation:** Pool recomputes HF using latest oracle data. If HF ≥ 1, the call reverts.
3. **Close Factor Enforcement:** `repayAmount` is capped at `min(requested, 50% × totalDebt)`.
4. **Debt Burn:** Stable or variable debt tokens are burned proportionally using the updated borrow index.
5. **Collateral Seizure:** Collateral worth `repayAmount × (1 + liquidationBonus)` is transferred to the liquidator (in aTokens if `receiveAToken = true`, otherwise as underlying).
6. **Event & Accounting:** `LiquidationCall` event is emitted; reserve indexes are updated so remaining users see new interest accrual states.
