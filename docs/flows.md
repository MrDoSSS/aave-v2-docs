# Operational Flows

## Context
This section walks through the critical end-to-end flows on Aave v2. The diagrams focus on validation order, index updates, and token mint/burn operations as implemented in `LendingPool.sol` and `LendingPoolCollateralManager.sol`.

## Deposit → aToken Mint
`LendingPool.deposit` is guarded by `whenNotPaused`. The pool accrues interest before minting to keep depositor shares consistent.

```mermaid
sequenceDiagram
    participant U as User
    participant LP as LendingPool
    participant VR as ValidationLogic
    participant RS as Reserve Data
    participant IRS as InterestRateStrategy
    participant aT as AToken

    U->>LP: deposit(asset, amount, onBehalfOf)
    note right of LP: modifier whenNotPaused
    LP->>VR: validateDeposit(reserve, amount)
    LP->>RS: updateState()
    LP->>RS: updateInterestRates(asset, aToken, liquidityAdded = amount)
    RS->>IRS: calculateInterestRates(...)
    LP->>aT: safeTransferFrom(user → aToken, amount)
    LP->>aT: mint(onBehalfOf, amount, liquidityIndex)
    LP-->>U: emit Deposit
```

## Borrow → Debt Accounting
`LendingPool.borrow` (also `whenNotPaused`) validates the post-borrow health factor, then mints the appropriate debt token, updates rates, and releases liquidity from the reserve’s aToken.

```mermaid
sequenceDiagram
    participant U as User
    participant LP as LendingPool
    participant PO as PriceOracle
    participant VR as ValidationLogic
    participant RS as Reserve Data
    participant IRS as InterestRateStrategy
    participant vD as VariableDebtToken
    participant sD as StableDebtToken
    participant aT as AToken

    U->>LP: borrow(asset, amount, rateMode, onBehalfOf)
    note right of LP: modifier whenNotPaused
    LP->>PO: fetch collateral & debt prices
    LP->>VR: validateBorrow(userAccountData, amount, rateMode)
    LP-->>U: revert if HF < 1 post-borrow
    LP->>RS: updateState()
    alt Variable rate
        LP->>vD: mint(user, onBehalfOf, amount, variableBorrowIndex)
    else Stable rate
        LP->>sD: mint(user, onBehalfOf, amount, currentStableRate)
    end
    LP->>RS: updateInterestRates(asset, aToken, liquidityTaken = amount)
    LP->>aT: transferUnderlyingTo(user, amount)
    LP-->>U: emit Borrow
```

## Liquidation Trigger & Settlement
`LendingPool.liquidationCall` carries a `whenNotPaused` modifier and delegates to `LendingPoolCollateralManager` via `delegatecall`. The manager enforces the fixed 50% close factor, burns the correct debt tokens, and transfers collateral with the liquidation bonus applied.

```mermaid
sequenceDiagram
    participant K as Liquidator
    participant LP as LendingPool
    participant CM as CollateralManager
    participant PO as PriceOracle
    participant RS as Reserve Snapshots
    participant vD as VariableDebtToken
    participant sD as StableDebtToken
    participant aT as AToken

    K->>LP: liquidationCall(collateralAsset, debtAsset, user, repayAmount, receiveAToken)
    note right of LP: modifier whenNotPaused
    LP->>CM: delegatecall liquidation logic
    CM->>PO: fetch prices & recompute HF(user)
    CM-->>K: revert if HF ≥ 1
    CM->>RS: cap repayAmount to CLOSE_FACTOR = 50%
    alt Variable debt outstanding
        CM->>vD: burn(amount / variableBorrowIndex)
    else Stable debt outstanding
        CM->>sD: burn(principal share)
    end
    CM->>RS: updateState() & updateInterestRates(collateralAsset, debtAsset)
    CM->>aT: burn user balance, transfer collateral (underlying or aTokens) to liquidator
    CM->>LP: return (NO_ERROR)
    LP-->>K: emit LiquidationCall
```

## Notes
- Each flow calls `reserve.updateState()` before minting or burning tokens so scaled balances reflect accrued interest.
- `setPoolPause(true)` blocks these entry points entirely. Freezing a reserve prevents new deposits/borrows but still allows repayments and liquidations.
- Flash-loan adapters (e.g., Uniswap/ParaSwap helpers) plug into the same flows by acting as flash-loan receivers; they must settle the flash debt plus premium within the transaction.
