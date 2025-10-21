# Operational Flows

## Context
This section captures the runtime sequences of the most common Aave v2 operations. Each Mermaid diagram mirrors the order of validations, state updates, and token movements in `LendingPool.sol` and its collateral manager.

## Deposit → aToken Mint
`LendingPool.deposit` (guarded by `whenNotPaused`) accrues reserve indexes before minting aTokens so depositor shares remain proportional.

```mermaid
sequenceDiagram
    participant U as User
    participant LP as LendingPool
    participant VR as ValidationLogic
    participant RS as Reserve Data
    participant IRS as InterestRateStrategy
    participant Tok as Asset ERC20
    participant aT as AToken

    U->>LP: deposit(asset, amount, onBehalfOf)
    note right of LP: whenNotPaused
    LP->>VR: validateDeposit(reserve, amount)
    LP->>RS: updateState()
    LP->>RS: updateInterestRates(asset, aToken, liquidityAdded = amount)
    RS->>IRS: calculateInterestRates(...)
    LP->>Tok: safeTransferFrom(user → aToken, amount)
    LP->>aT: mint(onBehalfOf, amount, liquidityIndex)
    LP-->>U: emit Deposit
```

## Withdraw → aToken Burn
`LendingPool.withdraw` burns aTokens, validates post-withdraw health, and sends underlying to the recipient (`whenNotPaused`).

```mermaid
sequenceDiagram
    participant U as User
    participant LP as LendingPool
    participant VR as ValidationLogic
    participant RS as Reserve Data
    participant aT as AToken
    participant Tok as Asset ERC20

    U->>LP: withdraw(asset, amount, to)
    note right of LP: whenNotPaused
    LP->>aT: balanceOf(user) → determine amountToWithdraw
    LP->>VR: validateWithdraw(asset, amountToWithdraw, userBalance, accountData)
    LP->>RS: updateState()
    LP->>RS: updateInterestRates(asset, aToken, liquidityTaken = amountToWithdraw)
    alt full balance withdrawn
        LP->>LP: clear collateral flag for reserve
    end
    LP->>aT: burn(user → to, amountToWithdraw, liquidityIndex)
    aT-->>Tok: transfer underlying to recipient
    LP-->>U: emit Withdraw
```

## Borrow → Debt Accounting
`LendingPool.borrow` validates the borrower’s health factor, mints stable or variable debt, updates indexes, and releases underlying from the reserve (`whenNotPaused`).

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
    note right of LP: whenNotPaused
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
    aT-->>U: transferUnderlyingTo(amount)
    LP-->>U: emit Borrow
```

## Repay → Debt Burn
`LendingPool.repay` burns stable or variable debt, increases reserve liquidity, and clears borrowing flags when balances reach zero (`whenNotPaused`).

```mermaid
sequenceDiagram
    participant P as Payer
    participant LP as LendingPool
    participant VR as ValidationLogic
    participant RS as Reserve Data
    participant vD as VariableDebtToken
    participant sD as StableDebtToken
    participant Tok as Asset ERC20
    participant aT as AToken

    P->>LP: repay(asset, amount, rateMode, onBehalfOf)
    note right of LP: whenNotPaused
    LP->>VR: validateRepay(reserve, amount, debts)
    LP->>RS: updateState()
    alt Variable debt
        LP->>vD: burn(onBehalfOf, paybackAmount, variableBorrowIndex)
    else Stable debt
        LP->>sD: burn(onBehalfOf, paybackAmount)
    end
    LP->>RS: updateInterestRates(asset, aToken, liquidityAdded = paybackAmount)
    LP->>Tok: safeTransferFrom(payer → aToken, paybackAmount)
    LP->>aT: handleRepayment(payer, paybackAmount)
    alt debt fully cleared
        LP->>LP: clear borrowing flag for reserve
    end
    LP-->>P: emit Repay
```

## Rate Mode Swap
`LendingPool.swapBorrowRateMode` switches a position between stable and variable debt without changing principal (`whenNotPaused`).

```mermaid
sequenceDiagram
    participant U as User
    participant LP as LendingPool
    participant VR as ValidationLogic
    participant RS as Reserve Data
    participant vD as VariableDebtToken
    participant sD as StableDebtToken

    U->>LP: swapBorrowRateMode(asset, newMode)
    note right of LP: whenNotPaused
    LP->>VR: validateSwapRateMode(reserve, userConfig, debts, newMode)
    LP->>RS: updateState()
    alt New mode = Variable
        LP->>sD: burn(user, stableDebt)
        LP->>vD: mint(user, user, stableDebt, variableBorrowIndex)
    else New mode = Stable
        LP->>vD: burn(user, variableDebt, variableBorrowIndex)
        LP->>sD: mint(user, user, variableDebt, currentStableRate)
    end
    LP->>RS: updateInterestRates(asset, aToken, liquidityAdded = 0, liquidityTaken = 0)
    LP-->>U: emit Swap
```

## Stable Rate Rebalance
`LendingPool.rebalanceStableBorrowRate` forces high-utilisation stable loans to the current stable rate to protect deposit yields (`whenNotPaused`).

```mermaid
sequenceDiagram
    participant K as Keeper
    participant LP as LendingPool
    participant VR as ValidationLogic
    participant RS as Reserve Data
    participant sD as StableDebtToken

    K->>LP: rebalanceStableBorrowRate(asset, user)
    note right of LP: whenNotPaused
    LP->>VR: validateRebalanceStableBorrowRate(reserve, asset, tokens, aToken)
    LP->>RS: updateState()
    LP->>sD: burn(user, stableDebt)
    LP->>sD: mint(user, user, stableDebt, currentStableBorrowRate)
    LP->>RS: updateInterestRates(asset, aToken, liquidityAdded = 0, liquidityTaken = 0)
    LP-->>K: emit RebalanceStableBorrowRate
```

## Liquidation Trigger & Settlement
`LendingPool.liquidationCall` delegates to `LendingPoolCollateralManager`, which enforces the 50% close factor and manages collateral transfers (`whenNotPaused`).

```mermaid
sequenceDiagram
    participant K as Liquidator
    participant LP as LendingPool
    participant CM as CollateralManager
    participant PO as PriceOracle
    participant dR as Debt Reserve
    participant cR as Collateral Reserve
    participant vD as VariableDebtToken
    participant sD as StableDebtToken
    participant aT as Collateral aToken
    participant TokD as Debt ERC20

    K->>LP: liquidationCall(collateralAsset, debtAsset, user, repayAmount, receiveAToken)
    note right of LP: whenNotPaused
    LP->>CM: delegatecall liquidation logic
    CM->>PO: fetch prices & recompute HF(user)
    CM-->>K: revert if HF ≥ 1
    CM->>dR: updateState()
    CM->>CM: cap repayAmount to CLOSE_FACTOR = 50%
    alt Variable debt outstanding
        CM->>vD: burn(user, amount/variableBorrowIndex)
    else Stable debt outstanding
        CM->>sD: burn(user, principal share)
    end
    CM->>dR: updateInterestRates(debtAsset, debtReserve.aToken, liquidityAdded = repayAmount)
    alt receive aTokens
        CM->>aT: transferOnLiquidation(user → liquidator, collateralAmount)
    else receive underlying
        CM->>cR: updateState()
        CM->>cR: updateInterestRates(collateralAsset, aToken, liquidityTaken = collateralAmount)
        CM->>aT: burn(user → liquidator, collateralAmount)
    end
    CM->>TokD: safeTransferFrom(liquidator → debtReserve.aToken, repayAmount)
    CM->>LP: return (NO_ERROR)
    LP-->>K: emit LiquidationCall
```

## Notes
- Every flow calls `reserve.updateState()` before token mint/burn to keep scaled balances accurate.
- `setPoolPause(true)` blocks all entry points; freezing a reserve stops new supply/borrow but still allows repayments and liquidations.
- Flash-loan adapters (Uniswap/ParaSwap helpers) interact with these flows by acting as flash-loan receivers, repaying principal plus premium within the same transaction.
